---
title: Calculating geo distances, but fast
---

A few days ago I faced, again, the problem of calculating the distance between two points over the surface of the Earth. This calculation involves some trigonometry a bit more complex than the planar one we all studied in high school: [spherical trigonometry](https://en.wikipedia.org/wiki/Spherical_trigonometry).

To calculate a distance between to coordinate sets exist two main algorithms: [haversine](https://en.wikipedia.org/wiki/Haversine_formula) and [Vincenty's](https://en.wikipedia.org/wiki/Vincenty%27s_formulae). The main difference is that the first one assumes Earth is a sphere while the second considers the surface to be an spheroid and is iterative.

As I was looking for a fast way to calculate the distances, the iterative approach was a no-go. But before start writing on my own the formulas I needed, I headed over [npm.com](https://www.npmjs.com/) searching for packages that were already providing this functionality. A quick [search by tag](https://www.npmjs.com/browse/keyword/haversine) returned just more than 10 packages.

But how to select a module? Unfortunately, the [npm website](https://www.npmjs.com/) does not provide stats on the packages list. The ratings provided by [npmsearch.com](http://npmsearch.com/?q=keywords:haversine) did not provide much more information to pick one either.

So I had to check one by one for clues on which one better matched my needs. But how? Easy: check every package's documentation and the code itself to select the best one. Well, not so easy... The code I saw in some packages was clearly inefficient so I started to revisit my decision of using an existing one.

But as I was looking for a fast implementations, I decided to run a Quick & Dirty (TM) benchmark before taking the final decision on doing my one package.

## Benchmarking packages

The overall idea of the benchmark was to generate thousands of coordinate pairs and feed each package with that list while measuring the time needed to crunch all the numbers. But in order to have one additional control point, also had to sum up all the distances to be sure the packages had a valid implementation.

The following code was used to generate the coordinate pairs:

```js
const COUNT = 50000;

const LAT_CENTER = -34.6;
const LAT_SPREAD = 10;
const LON_CENTER = -58.4;
const LON_SPREAD = 10;

function generateCoord() {
  return {
    lat: Math.random() * LAT_SPREAD + LAT_CENTER - 90,
    lon: Math.random() * LON_SPREAD + LON_CENTER - 180
  };
}

function generateCoordPairs(count) {
  return new Array(count).fill(null).map(() => ({
    from: generateCoord(),
    to: generateCoord()
  }))
}

const testData = generateCoordPairs(COUNT);
```

The variable `COUNT` controls how many pairs are generated while the variables `LAT_CENTER`, `LAT_SPREAD`, `LON_CENTER` and `LON_SPREAD` generate the dispersion of the data points: all around the globe or keep them between certain ranges/region. The current setup generates points +/-5 degrees around Buenos Aires, Argentina, covering most of the country.

The main difficulty of testing every package was the fact that each one had a different interface. Some packages returned a function accepting a pair of arrays, some required two coordinate objects and others returned more than one function, so had to pick the right one. In addition, some packages returned the distances in meters, others in kilometers and so on, adding a bit of additional complexity to the test.

So, the way to solve the issue was to code a generic runner that would feed the test data to each package, gather the results and print a report and to provide an adapter to interface each package's own API to the common API expected by the generic runner, including proper distance units normalization.

The runner looked like this:

```js
function bench(libName, fn, factor) {
  const lib = require(libName);

  const start = Date.now();
  const distances = testData.map(item => fn(lib, item.from, item.to));
  const time = Date.now() - start;

  return {
    libName,
    time,
    dist: Math.floor(distances.reduce((p, c) => +p + +c) * factor)
  };
}
```

and one of the benchmarks using an adapter functions looked like:

```js
// gps-distance package:
bench('gps-distance', function (lib, from, to) {
  return lib(from.lat, from.lon, to.lat, to.lon);
}, 1);
```

Finally, the report sorted the results and normalized the times and distances to the fastest one:

```js
results.sort((a, b) => a.time - b.time).map((item, ix, array) => ({
  name: item.libName,
  time: Math.floor(item.time / array[0].time * 100) + '%',
  dist: Math.floor(item.dist / array[0].dist * 100) + '%'
}))
```

The initial results confirmed some packages had much better implementations than others:

```
$ node benchmark.js
[ { name: 'gps-distance', time: '100%', dist: '100%' },
  { name: 'haversine-distance', time: '108%', dist: '100%' },
  { name: 's-haversine', time: '120%', dist: '100%' },
  { name: 'geodesy', time: '127%', dist: '100%' },
  { name: 'haversine', time: '136%', dist: '100%' },
  { name: 'geodist', time: '194%', dist: '99%' },
  { name: 'coordist', time: '229%', dist: '100%' },
  { name: 'node-geo-distance', time: '405%', dist: '100%' },
  { name: 'jeyo-distans', time: '624%', dist: '100%' } ]
```

The difference between the fastest and the slowest was 6x! But even the best ones had room for improvement.

## Fast haversine

On top of some obvious improvements, like avoid repeating calculations each time the distance function is called, or to inline the power calculations instead of calling a function, my own use case allowed me to take some additional shortcuts.

The full algorithm involves calculating 7 expensive trigonometric functions (4 `sin`, 2 `cos`, 1 `atan`), to pows and two square roots and looks something like this:

```js
// (mean) radius of Earth (meters)
const R = 6378137;

function toRad(n) {
  return n * Math.PI / 180;
}

function distance(a, b) {
  const dLat = toRad(b.lat - a.lat);
  const dLon = toRad(b.lon - a.lon);
  const aLat = toRad(a.lat);
  const bLat = toRad(b.lat);

  const f = Math.pow(Math.sin(dLat / 2), 2) + (Math.pow(Math.sin(dLon / 2), 2) * Math.cos(aLat) * Math.cos(bLat));
  const c = 2 * Math.atan2(Math.sqrt(f), Math.sqrt(1 - f));

  return R * c;
}
```

Since I was just dealing with small distances and

> Trigonometry lesson #42: lim x->0 sin(x) = x

I got rid of the `sin` calculations. On top of that, the `cos` of two closer latitudes is aproximately the `cos` of its average. That allowed me to remove one `cos` and assume some additional error.

Finally, the two `pow` were inlined, so no need to call an external generic power function if only need to multiply a number by itself once.

The final algorithm, with some additional inlining to optimize even more the code, looked like this:

```js
// (mean) radius of Earth (meters)
const R = 6378137;
const PI_360 = Math.PI / 360;

function dist(a, b) {
  const cLat = Math.cos((a.lat + b.lat) * PI_360);
  const dLat = (b.lat - a.lat) * PI_360;
  const dLon = (b.lon - a.lon) * PI_360;

  const f = dLat * dLat + cLat * cLat * dLon * dLon;
  const c = 2 * Math.atan2(Math.sqrt(f), Math.sqrt(1 - f));

  return R * c;
}
```

Running the benchmark again showed the improvements:

```
$ node benchmark.js
[ { name: './fastHaversine', time: '100%', dist: '100%' },
  { name: 'gps-distance', time: '137%', dist: '99%' },
  { name: 'haversine-distance', time: '151%', dist: '99%' },
  { name: 'geodesy', time: '165%', dist: '99%' },
  { name: 's-haversine', time: '166%', dist: '99%' },
  { name: 'haversine', time: '176%', dist: '99%' },
  { name: 'geodist', time: '254%', dist: '99%' },
  { name: 'coordist', time: '297%', dist: '100%' },
  { name: 'node-geo-distance', time: '522%', dist: '99%' },
  { name: 'jeyo-distans', time: '816%', dist: '99%' } ]

```

Overall, almost 30% more speed with ~1% of error!!!

Take a look at the whole code, including the benchmark at the `fast-haversine` [GitHub repo](https://github.com/gabmontes/fast-haversine) and [`npm package`](https://www.npmjs.com/package/fast-haversine).

Enjoy!

## Conclusions

Some lessons learned from this process:

- Check out there before implementing an algorithm. It's almost certain that someone has already done it for you before.
- Don't get the first package that appears in the `npm` listings or is mentioned elsewhere. Check the documentation, look at the code, if it is being actively maintained or not before taking a decision.
- If you are unsure about which one to pick, benchmark against your actual use case.
- Only then and if no one looks good enough, code your own. And remember to publish it and let everyone know!
