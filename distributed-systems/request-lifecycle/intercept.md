## Where I Noticed the Gap
**Two concepts:**
* **Old (Deprecated)** → Initially, I wrote interception documentation while working on `prolepsis`'s GitHub Callback tests, but it got deleted—since it was pure nightmare fuel. Its suggestion was "`Cmd+Click` 'Go to Definition' until you hit the Linux kernel"—which was bad. The "OAuth2 Token Exchange" was already difficult to intercept; by spamming my old strategy, you would say *"Mmmmh... Now, I'll start creating a test, and I'll intercept the `http.Do` command"* at 2:23 PM, and only start coding at 8:43 AM the next day—because that's when you finally figured out what to intercept.
* **New** → Simply look at what you want to intercept, hover your cursor over it, and see what it expects (a method that takes `x` parameters and returns `y`). Then, create an interface where you declare that exact same signature. After that, swap in the interface instead of the concrete struct (`*http.Client`, `*kgo.Client`).



## The Problem
* Usually, when we write a test, we don't want our function to actually hit the network when executed. For example, if you test a function that sends a request to the Twitter API, you don't want it to execute for real using your live API key—because the day you run that test 1,000 times, it will drain all the credits on your API key.

## Underlying Concept / How It Works
* I'll explain how everything works by splitting it into phases:
* **Phase 1** → First, check the signature of the function you want to intercept. For example, when you hover your cursor over `http.Client.Do(req)`, it shows `func (c *http.Client) Do(req *http.Request) (*http.Response, error)`. See? From this we know → it's a method, it expects a `*http.Request` as a parameter, and it returns a `*http.Response` and an `error`. Now that we have this, move to the next phase.
* **Phase 2** → Create an interface (you pick the name) and assign that exact signature as its contract (stripping away `func` and the receiver `(c *http.Client)`), leaving `Do(req *http.Request) (*http.Response, error)`. After this, move to the next phase.
* **Phase 3** → Inside the production code, change the struct field's type from `*http.Client` to the interface, since the interface will now act in place of that struct, and the method becomes the contract.
* **Phase 4** → Inside the test, create a mock struct (doesn't matter if it's empty or holds fields) and attach your mock method to it—as long as it satisfies the contract.

## Workflow / Lifecycle (Diagram)

//

## Example Code and Explanation
* **Inside `port.go`:**
```go
package domain

import (
    "net/http"
)

type HttpDo interface {
    Do(req *http.Request) (*http.Response, error)
}
```
* * Just as stated, here we define the contract for what we want to mock (we strip away `func` and the receiver, since the interface itself acts as the receiver).
* **Inside `currency_track`:**
```go
package currency

import (
	"context"
	"net/http"
	"tsubaki/domain"

	"github.com/riverqueue/river"
)

type CurrencyTrackerWorker struct {
    river.WorkerDefaults[CurrencyTrackerArgs]
    HttpClient HttpDo // Notice? We use our interface now
}

func (c *CurrencyTrackWorker) Work(ctx context.Context, job *river.Job[CurrencyTrackArgs]) error {

    // Later in the code

    resp, err := c.HttpClient.Do(req) // <-- "HttpClient" can now be anything we want, and "Do" follows our contract—giving us full power to swap it as we wish
}
```
* * Notice? `CurrencyTrackerWorker` no longer knows or cares if `HttpClient` is real or fake—it just calls `.Do(req)` and trusts whatever was passed into that slot.
* **Inside `main.go`:**
```go
package main

func main() {
    currencyWorker := &currency.CurrencyTrackWorker{
        HttpClient: &httpClient, // <-- See? In main we pass a real http.Client—while in the test we just pass a mock struct with our own Do method
		Rates:      &domain.CurrencyStore{},
    }
}
```
* * In production (`main.go`), we plug in the real `&httpClient` so real network calls happen. When we move to `_test.go`, we keep this exact same setup—except we swap `&httpClient` for our mock struct and return whatever fake response we want.