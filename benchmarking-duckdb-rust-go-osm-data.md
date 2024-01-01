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
use std::{time::Instant, ops::Div, collections::HashMap};
use osmpbf::{ElementReader, Element};
use geo::HaversineDistance;
use geo_types::Point;

fn calculate_avg_distance(data: HashMap<&str, Point>) -> f64 {
    
    let mut health_centres:HashMap<&str, Point> = HashMap::new();
    let mut pharmacies:HashMap<&str, Point> = HashMap::new();
    let mut distances:HashMap<&str, f64> = HashMap::new();

    for (key, value) in data.iter() {
        if key.contains("health_care") {
            health_centres.insert("health_care", *value);
        } else if key.contains("pharmacy") {
            pharmacies.insert("pharmacy", *value);
        }
    }

    for (_, hc_val) in health_centres.iter() {
        for (_, p_val) in pharmacies.iter() {
            let distance = hc_val.haversine_distance(&p_val);
            distances.insert("d", distance);
        }
    }

    let mut sum = 0.0;
    for (_, distance) in distances.iter() {
        sum += distance
    }
    let count = distances.len();

    let average = sum / count as f64;

    average

}

fn open_osmpbf(area: &str) -> HashMap<&str, Point> {
    
    let filepath = format!("../../data/{}-latest.osm.pbf", area);
    let reader = ElementReader::from_path(filepath).expect("Error opening file");

    let features = reader.par_map_reduce(
        |element| {
            let mut health_centre: Option<Point> = None; 
            let mut data: HashMap<&str, geo::Point> = HashMap::new();
            let mut pharmacy: Option<Point> = None; 
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
                    data.insert("health_centre", hc);
                    data.insert("pharmacy", p);
                    data
                },
                (Some(hc), None) => {
                    data.insert("health_centre", hc);
                    data
                },
                (None, Some(p)) => {
                    data.insert("pharmacy", p);
                    data
                }
                (None, None) => data
            }
        },
        || HashMap::<&str, geo::Point>::new(),
        |mut a, b| {
            a.extend(&b);
            a
        },
    ).expect("Error reading file");


    features

}

fn main() {
    let now = Instant::now();
    let area = "planet";
    let data = open_osmpbf(area);
    let avg_distance = calculate_avg_distance(data);
    println!("Average distance of Pharmacies to Health Centres in {}: {} kilometers", area, avg_distance.div(1000.00).ceil());
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
	//"runtime/debug"
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
	start := time.Now()
    sum := 0.0
	var wg sync.WaitGroup
	var mu sync.Mutex

    for _, valueInt := range arr {
		wg.Add(1)
		go func(valueInt float64) {
			defer wg.Done()
			mu.Lock()
			sum += valueInt
			mu.Unlock()
		}(valueInt)
    }
	fmt.Println(fmt.Sprintf("Time taken suming up: %f", time.Since(start).Seconds()))
    return sum
}

func calculateAvgDistance(g1 []orb.Point, g2 []orb.Point) float64 {
	start := time.Now()

	//var distances []float64
	var wg sync.WaitGroup
	var mu sync.Mutex
	var distance float64
	var count int

	for _, point_a := range g1 {
		wg.Add(1)
		go func(point_a orb.Point) {
			defer wg.Done()
			for _, point_b := range g2 {
				mu.Lock()
				_ = point_b
				//distances = append(distances, geo.DistanceHaversine(point_a, point_b))
				distance += geo.DistanceHaversine(point_a, point_b) 
				count += 1
				mu.Unlock()
			}
		}(point_a)
	}

	wg.Wait()

	fmt.Println(fmt.Sprintf("Time taken calculating distance: %f", time.Since(start).Seconds()))

	avg := distance / float64(count)


	return avg

}

func main() {
	//debug.SetMemoryLimit(8 * 1024 * 1024 * 1024)
	start := time.Now()
	area := "planet"
	hc, p := openOsmpbf(area)
	avgDistance := calculateAvgDistance(hc, p)
    fmt.Println(fmt.Sprintf("Average distance of Pharmacies to Health Centres in %s: %d kilometers", area, int64(math.RoundToEven(avgDistance)) / 1000))
	fmt.Println(fmt.Sprintf("Time taken: %f", time.Since(start).Seconds()))
}
```
