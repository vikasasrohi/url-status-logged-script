package main

import (
	"bufio"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
	"sync"
	"time"
)

// URLStatus represents the status of a URL check.
type URLStatus struct {
	URL       string    json:"url"
	Status    string    json:"status"
	Timestamp time.Time json:"timestamp"
}

func checkURL(url string, results chan<- URLStatus, wg *sync.WaitGroup) {
	defer wg.Done()

	_, err := http.Get(url)
	status := "UP"
	if err != nil {
		status = "DOWN"
	}
	// fmt.Println(*resp)
	results <- URLStatus{
		URL:       url,
		Status:    status,
		Timestamp: time.Now(),
	}
}

func main() {
	// Read URLs from the file
	urls, err := readURLsFromFile("urls.txt")
	if err != nil {
		fmt.Printf("Error reading URLs from file: %s\n", err)
		return
	}

	// Set up channels and wait group
	results := make(chan URLStatus, len(urls))
	var wg sync.WaitGroup

	// Perform URL checks concurrently
	for _, url := range urls {
		wg.Add(1)
		go checkURL(url, results, &wg)
	}

	// Close the results channel once all checks are done
	go func() {
		wg.Wait()
		close(results)
	}()

	// Process results and log them to a JSON file
	var statuses []URLStatus
	for result := range results {
		statuses = append(statuses, result)
	}

	logResults(statuses)
}

func readURLsFromFile(filePath string) ([]string, error) {
	file, err := os.Open(filePath)
	if err != nil {
		fmt.Println(err)
	}
	defer file.Close()

	urls := []string{}
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		// fmt.Println(scanner.Text())
		urls = append(urls, scanner.Text())
	}

	if err := scanner.Err(); err != nil {
		fmt.Println(err)
	}

	return urls, nil
}

func logResults(statuses []URLStatus) {
	var lock sync.Mutex
	content, err := json.Marshal(statuses)
	if err != nil {
		fmt.Println(err)
	}
	lock.Lock()
	defer lock.Unlock()
	err = ioutil.WriteFile("userfile.json", content, 0644)
	if err != nil {
		fmt.Printf("Error creating output file: %s\n", err)
	}
	fmt.Println("Results logged to results.json")
}
