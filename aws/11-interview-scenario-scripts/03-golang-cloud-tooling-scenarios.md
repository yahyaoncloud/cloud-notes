# Go (Golang) Cloud Tooling Interview Scenarios: Practical Scripts, Questions & Answers

Go is the lingua franca of cloud infrastructure tooling (Docker, Kubernetes, Terraform, Prometheus, Envoy, and Helm are all written in Go). In cloud engineering interviews, Go scripting questions test concurrency (goroutines and channels), CLI engineering, and high-throughput AWS API automation using the official AWS SDK for Go v2.

---

## Scenario 1: High-Concurrency Cloud Endpoint Health Checker

### The Interview Question
"You need to verify the availability of 200 microservice endpoints across private VPC networks. Writing this sequentially in Python takes over 2 minutes due to connection timeouts. Write a concurrent CLI tool in Go that accepts a list of URLs, validates their HTTP status codes concurrently using a worker pool of 20 goroutines, enforces a strict per-request timeout of 3 seconds, and prints a formatted summary of healthy vs failed endpoints."

### Production Go Implementation

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"sync"
	"time"
)

// CheckResult encapsulates the outcome of an endpoint probe
type CheckResult struct {
	URL        string
	StatusCode int
	Duration   time.Duration
	Err        error
}

// worker processes URLs from a shared jobs channel
func worker(ctx context.Context, id int, jobs <-chan string, results chan<- CheckResult, wg *sync.WaitGroup) {
	defer wg.Done()

	// Reusable HTTP client with keep-alives and strict timeouts
	client := &http.Client{
		Timeout: 3 * time.Second,
	}

	for url := range jobs {
		select {
		case <-ctx.Done():
			results <- CheckResult{URL: url, Err: ctx.Err()}
			return
		default:
			start := time.Now()
			req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
			if err != nil {
				results <- CheckResult{URL: url, Err: err, Duration: time.Since(start)}
				continue
			}

			resp, err := client.Do(req)
			duration := time.Since(start)

			if err != nil {
				results <- CheckResult{URL: url, Err: err, Duration: duration}
			} else {
				results <- CheckResult{URL: url, StatusCode: resp.StatusCode, Duration: duration}
				_ = resp.Body.Close()
			}
		}
	}
}

func main() {
	endpoints := []string{
		"https://aws.amazon.com",
		"https://github.com",
		"https://httpbin.org/status/200",
		"https://httpbin.org/status/503",
		"https://invalid-non-existent-endpoint.internal",
	}

	const numWorkers = 5
	jobs := make(chan string, len(endpoints))
	results := make(chan CheckResult, len(endpoints))
	var wg sync.WaitGroup

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// Launch worker pool
	for w := 1; w <= numWorkers; w++ {
		wg.Add(1)
		go worker(ctx, w, jobs, results, &wg)
	}

	// Feed jobs into the channel
	for _, url := range endpoints {
		jobs <- url
	}
	close(jobs)

	// Wait for workers to finish in a separate goroutine
	go func() {
		wg.Wait()
		close(results)
	}()

	// Aggregate and display results
	fmt.Printf("%-45s | %-10s | %-12s | %s\n", "ENDPOINT", "STATUS", "LATENCY", "OUTCOME")
	fmt.Println("-----------------------------------------------------------------------------------------")

	for res := range results {
		if res.Err != nil {
			fmt.Printf("%-45s | %-10s | %-12v | ERROR: %v\n", res.URL, "FAIL", res.Duration.Round(time.Millisecond), res.Err)
		} else {
			fmt.Printf("%-45s | HTTP %-5d | %-12v | OK\n", res.URL, res.StatusCode, res.Duration.Round(time.Millisecond))
		}
	}
}
```

### Interview Discussion Points
* **Why Worker Pools over Unbounded Goroutines:** Spawning 10,000 goroutines without a pool can exhaust socket file descriptors (`too many open files`) or trigger local port exhaustion. A bounded worker pool controls concurrency while consuming minimal CPU/RAM.
* **Context Propagation:** Passing `context.Context` ensures graceful cancellation if the overall CLI tool is interrupted or times out.

---

## Scenario 2: High-Throughput S3 Batch Object Processor (AWS SDK for Go v2)

### The Interview Question
"Write a Go utility using the AWS SDK for Go v2 that connects to an S3 bucket, retrieves all objects matching a prefix `logs/raw/`, and checks if any object has a storage class of `STANDARD` that is older than 30 days, logging candidates for tiering."

### Production Go Implementation

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/aws/aws-sdk-go-v2/aws"
	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/s3"
	s3types "github.com/aws/aws-sdk-go-v2/service/s3/types"
)

func auditS3StorageTiering(ctx context.Context, bucketName, prefix string, ageThresholdDays int) error {
	// Load default AWS credentials and region from ~/.aws or environment
	cfg, err := config.LoadDefaultConfig(ctx, config.WithRegion("us-east-1"))
	if err != nil {
		return fmt.Errorf("unable to load AWS SDK config: %w", err)
	}

	client := s3.NewFromConfig(cfg)
	cutoffDate := time.Now().AddDate(0, 0, -ageThresholdDays)

	// Use official SDK paginator for ListObjectsV2
	paginator := s3.NewListObjectsV2Paginator(client, &s3.ListObjectsV2Input{
		Bucket: aws.String(bucketName),
		Prefix: aws.String(prefix),
	})

	var eligibleCount int
	var eligibleBytes int64

	fmt.Printf("Scanning s3://%s/%s for Standard objects older than %d days...\n", bucketName, prefix, ageThresholdDays)

	for paginator.HasMorePages() {
		page, err := paginator.NextPage(ctx)
		if err != nil {
			return fmt.Errorf("failed to retrieve S3 page: %w", err)
		}

		for _, obj := range page.Contents {
			// Objects in STANDARD storage class
			isStandard := obj.StorageClass == "" || obj.StorageClass == s3types.ObjectStorageClassStandard
			if isStandard && obj.LastModified.Before(cutoffDate) {
				eligibleCount++
				eligibleBytes += *obj.Size
				fmt.Printf("[CANDIDATE] Key: %-40s | Size: %8d KB | Modified: %s\n",
					*obj.Key, *obj.Size/1024, obj.LastModified.Format("2006-01-02"))
			}
		}
	}

	fmt.Printf("\nAudit complete. Found %d objects (%d MB) eligible for Glacier/IA transition.\n",
		eligibleCount, eligibleBytes/(1024*1024))
	return nil
}

func main() {
	ctx := context.TODO()
	if err := auditS3StorageTiering(ctx, "my-enterprise-data-lake-2026", "logs/raw/", 30); err != nil {
		log.Fatalf("Fatal error: %v", err)
	}
}
```

### Interview Discussion Points
* **AWS SDK v2 Architecture:** Highlight that AWS SDK for Go v2 uses modular Go modules (`service/s3`, `config`), context-driven function calls, and strongly typed enum definitions (`s3types.ObjectStorageClassStandard`).
