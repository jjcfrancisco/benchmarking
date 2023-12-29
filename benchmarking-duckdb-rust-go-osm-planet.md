# Benchmarks:

## Test 1: count features
The test consists of opening the appropriate osm.pbf file and count the number of features.

| area      | features   | duckdb  | rust    | go      |
| --------- | ---------- | ------- | ------- | ------- |
| Andalucía | 17189833   | 0.421   | 0.472   | 1.054   |
| Spain     | 162478451  | 3.770   | 4.243   | 9.269   |
| Europe    | 3695049613 | 94.6    | 109.3   | 218.5   |
| Planet    | 9800308007 | 134.6   | 281.8   | 700.393 |

> Note: time profiling in miliseconds

**Duckdb code - test 1:**
```sql
INSTALL spatial;
LOAD spatial;
EXPLAIN ANALYZE SELECT count(*) AS features FROM st_readosm('planet-latest.osm.pbf');
```

**Rust code - test 1:**
```rust
use osmpbf::{ElementReader, Element};
use std::time::Instant;

fn open_osmpbf() {
    let now = Instant::now();
    let filepath = format!("../../data/{}-latest.osm.pbf", area);
    let reader = ElementReader::from_path(filepath).expect("Error opening file");

    let features = reader.par_map_reduce(
        |element| {
            match element {
                Element::Node(_) => 1,
                Element::Way(_) => 1,
                Element::Relation(_) => 1,
                Element::DenseNode(_) => 1,
            }
        },
        || 0_u64,
        |a, b| a + b
    ).expect("Error reading file");

    println!("Features: {}", features);
    println!("Time taken: {}", now.elapsed().as_secs_f32());

}

fn main() {
    open_osmpbf("planet")
}
```

**Go code - test 1:**
```go
package main

import (
	"github.com/paulmach/osm/osmpbf"
	"github.com/paulmach/osm"
	"runtime"
	"time"
	"context"
	"os"
	"fmt"
)

func OpenOsmpbf() {

	var features int

	file, err := os.Open("europe-latest.osm.pbf")
	if err != nil {
		panic(err)
	}
	defer file.Close()

    // runtime.GOMAXPROCS set to 8 cores
	scanner := osmpbf.New(context.Background(), file, runtime.GOMAXPROCS(8))
	defer scanner.Close()

	for scanner.Scan() {

		obj := scanner.Object()

		switch obj.(type) {
		case *osm.Node: features += 1
		case *osm.Way: features += 1
		case *osm.Relation: features += 1
		}
	}

	if err := scanner.Err(); err != nil {
		panic(err)
	}

	fmt.Println(features)
}

func main() {
	start := time.Now()
	OpenOsmpbf()
	elapsed := time.Since(start)
	fmt.Println(elapsed)
}
```


## Test 2: average distance
The test builds upon test 1, calculates the average distances between health centres and pharmacies in a given osm.pbf

| area      | features   | duckdb  | rust    | go      | kms  |
| --------- | ---------- | ------- | ------- | ------- | ---- |
| Andalucía | 17189833   | 0.90s   | 0.55    | 1.15    | 127  |
| Spain     | 162478451  | 8.40s   | 5.02    | 11.05   | 450  |
| Europe    | 3695049613 | 220.47s | 142.43  | 370.45  | 1422 |
| Planet    | 9800308007 | 264.82  | 00000   | 00000   |

> Note: time profiling in miliseconds

**Duckdb code - test 2:**
```sql
-- I had to implement the haversine formula manually
-- as Duckdb was not producing the expected results
INSTALL spatial;
LOAD spatial;
EXPLAIN ANALYZE WITH osm AS (SELECT tags, lon, lat
FROM st_readosm('../data/europe-latest.osm.pbf')
WHERE lon IS NOT NULL
AND lat IS NOT NULL),
hc AS (SELECT * FROM osm WHERE list_contains(tags['healthcare'], 'centre')),
p AS (SELECT * FROM osm WHERE list_contains(tags['amenity'], 'pharmacy'))
SELECT CEIL(AVG(6371 * acos(cos(radians(p.lat)) * cos(radians(hc.lat)) * cos(radians(hc.lon) - radians(p.lon)) + sin(radians(p.lat)) * sin(radians(hc.lat))))) AS distance
FROM p, hc;
```

**Rust code - test 2:**
```rust
use std::{time::Instant, ops::Div};
use osmpbf::{ElementReader, Element};
use rayon::prelude::*;
use geo::HaversineDistance;
use geo_types::Point;

fn calculate_avg_distance(geom1: Vec<Point>, geom2: Vec<Point>) -> f64 {

    let distances: Vec<f64> = geom1.iter()
                                   .flat_map(|g1| {
                                       geom2.iter()
                                            .flat_map(move |g2| {
                                                Some(g1.haversine_distance(g2))
                                            })
                                   }).collect();

    let sum: f64 = distances.iter().sum();
    let count = distances.iter().len();

    let average = sum / count as f64;

    average

}

fn open_osmpbf(area: &str) -> (Vec<Point>, Vec<Point>) {
    
    let filepath = format!("../../data/{}-latest.osm.pbf", area);
    let reader = ElementReader::from_path(filepath).expect("Error opening file");

    let features = reader.par_map_reduce(
        |element| {
            let mut health_centre: Option<Point> = None; 
            let mut health_centres: Vec<geo::Point> = vec![];
            let mut pharmacy: Option<Point> = None; 
            let mut pharmacies: Vec<geo::Point> = vec![];
            let result:(Option<Point>, Option<Point>) = match element {
                Element::Node(n) => {
                    for (key, value) in n.tags() {
                        if key == "healthcare" && value == "centre" {
                            health_centre = Some(Point::new(n.lon(), n.lat()));
                        } else if key == "amenity" && value == "pharmacy" {
                            pharmacy = Some(Point::new(n.lon(), n.lat()));
                        }
                    }
                    (health_centre, pharmacy)
                },
                Element::DenseNode(dn) => {
                    for (key, value) in dn.tags() {
                        if key == "healthcare" && value == "centre" {
                            health_centre = Some(Point::new(dn.lon(), dn.lat()));
                        } else if key == "amenity" && value == "pharmacy" {
                            pharmacy = Some(Point::new(dn.lon(), dn.lat()));
                        }
                    }
                    (health_centre, pharmacy)
                },
                Element::Relation(_) => (None, None),
                Element::Way(_) => (None, None),
            };

            match (result.0, result.1) {
                (Some(hc), Some(p)) => {
                    health_centres.push(hc);
                    pharmacies.push(p);
                    (health_centres, pharmacies)
                },
                (Some(hc), None) => {
                    health_centres.push(hc);
                    (health_centres, pharmacies)
                },
                (None, Some(p)) => {
                    pharmacies.push(p);
                    (health_centres, pharmacies)
                }
                (None, None) => (health_centres, pharmacies)
            }
        },
        || (Vec::<geo::Point>::new(), Vec::<geo::Point>::new()),
        |mut a, b| {
            a.0.extend_from_slice(&b.0);
            a.1.extend_from_slice(&b.1);
            (a.0, a.1)
        },
    ).expect("Error reading file");


    features

}

fn main() {
    let now = Instant::now();
    let area = "planet";
    let hc_p = open_osmpbf(area);
    println!("Health centres: {}", hc_p.0.len());
    println!("Pharmacies: {}", hc_p.1.len());

    let avg_distance = calculate_avg_distance(hc_p.0, hc_p.1);
    println!("Average distance of Pharmacies to Health Centres: {} kilometers", avg_distance.div(1000.00).ceil());
    println!("Time taken: {}", now.elapsed().as_secs_f32());
}
```

**Go code - test 2:**
```go
package main

import (
	"github.com/paulmach/osm/osmpbf"
	"github.com/paulmach/osm"
	"github.com/paulmach/orb"
	"github.com/paulmach/orb/geo"
	"runtime"
	"time"
	"context"
	"os"
	"fmt"
	"sync"
	"math"
)

func openOsmpbf(fname string) ([]orb.Point, []orb.Point) {

	file, err := os.Open(fmt.Sprintf("../data/%s-latest.osm.pbf", fname))
	if err != nil {
		panic(err)
	}
	defer file.Close()

	// The third parameter is the number of parallel decoders to use.
	scanner := osmpbf.New(context.Background(), file, runtime.GOMAXPROCS(8))
	defer scanner.Close()

	var health_centres []orb.Point
	var pharmacies []orb.Point

	for scanner.Scan() {
		obj := scanner.Object()
		switch o := obj.(type) {
		case *osm.Node:
			tags := o.TagMap()
			if amenity, ok := tags["healthcare"]; ok && amenity == "centre" {
				health_centres = append(health_centres, o.Point())
			} else if amenity, ok := tags["amenity"]; ok && amenity == "pharmacy" {
				pharmacies = append(pharmacies, o.Point())
			}
		case *osm.Way: 
		case *osm.Relation: 
		}
	}

	if err := scanner.Err(); err != nil {
		panic(err)
	}

	return health_centres, pharmacies

}

func sum(arr []float64) float64 {
    sum := 0.0
    for _, valueInt := range arr {
        sum += valueInt
    }
    return sum
}

func calculateAvgDistance(g1 []orb.Point, g2 []orb.Point) float64 {

	var distances []float64
	var wg sync.WaitGroup
	var mu sync.Mutex

	for _, point_a := range g1 {
		wg.Add(1)
		go func(point_a orb.Point) {
			defer wg.Done()
			for _, point_b := range g2 {
				mu.Lock()
				distances = append(distances, geo.DistanceHaversine(point_a, point_b))
				mu.Unlock()
			}
		}(point_a)
	}

	wg.Wait()

	result := sum(distances)

	avg := (result) / (float64(len(distances)))

	return avg

}

func main() {
	start := time.Now()
	hc, p := openOsmpbf("europe")
    fmt.Println(fmt.Sprintf("Health centres: %d", len(hc)))
    fmt.Println(fmt.Sprintf("Pharmacies: %d", len(p)))
	avgDistance := calculateAvgDistance(hc, p)
    fmt.Println(fmt.Sprintf("Average distance of Pharmacies to Health Centres: %d kilometers", int64(math.RoundToEven(avgDistance)) / 1000))
	elapsed := time.Since(start)
	fmt.Println(fmt.Sprintf("Time taken: %f", elapsed.Seconds()))
}
```