# Go Tooling And Modern Idioms

## Module And Toolchain

- Set the language version in `go.mod` (`go 1.xx`) and let the `toolchain` directive pin the exact toolchain for reproducible builds.
- Commit `go.mod` and `go.sum`; verify with `go mod verify`; keep `go mod tidy` clean in CI (fail if it produces a diff).
- Track developer tools in the module with `tool` directives (`go get -tool example.com/cmd/stringer`, run with `go tool stringer`) instead of untracked `go install` steps.

## The Required Gate

```bash
gofmt -l .                   # must print nothing (or use `goimports`)
go vet ./...
golangci-lint run            # staticcheck, errcheck, govet, ineffassign, gosec, ...
go test -race -shuffle=on ./...
govulncheck ./...            # known vulnerabilities in dependencies and stdlib
```

- Run tests with `-race` in CI; data races are bugs even when tests pass without it.
- Use `-shuffle=on` to expose order dependence and `-count=1` to bypass the test cache when needed.

## Idioms Worth Using

```go
package main

import (
	"errors"
	"fmt"
	"iter"
	"log/slog"
	"maps"
	"slices"
)

type Item struct {
	Name  string
	Price int
}

// range over int (Go 1.22)
func countTo(n int) []int {
	out := make([]int, 0, n)
	for i := range n {
		out = append(out, i)
	}
	return out
}

// iterators (Go 1.23): range over a function
func Positive(items []Item) iter.Seq[Item] {
	return func(yield func(Item) bool) {
		for _, it := range items {
			if it.Price > 0 && !yield(it) {
				return
			}
		}
	}
}

// errors.Join aggregates several failures; errors.Is / errors.As inspect wrapped errors
func validate(it Item) error {
	var errs []error
	if it.Name == "" {
		errs = append(errs, errors.New("name is required"))
	}
	if it.Price < 0 {
		errs = append(errs, fmt.Errorf("price %d: %w", it.Price, errors.ErrUnsupported))
	}
	return errors.Join(errs...)
}

func main() {
	logger := slog.Default() // structured logging in the standard library
	items := []Item{{"a", 3}, {"", -1}, {"c", 0}}

	for it := range Positive(items) {
		logger.Info("positive item", "name", it.Name, "price", it.Price)
	}

	names := slices.Sorted(maps.Keys(map[string]int{"b": 1, "a": 2})) // slices and maps packages
	fmt.Println(countTo(3), names, validate(items[1]) != nil)
}
```

- Use `log/slog` for structured logs, `slices`/`maps` for collection helpers, and `errors.Is/As/Join` for error inspection.
- Wrap errors with context: `fmt.Errorf("load user %d: %w", id, err)`. Return errors; do not panic for expected failures.
- Pass `context.Context` as the first parameter of calls that block or do I/O, and respect cancellation.
- Accept interfaces, return concrete types; define small interfaces where they are consumed.
- Do not start goroutines without a way to stop and wait for them (`errgroup`, `sync.WaitGroup`, context cancellation).

## Testing

- Table-driven tests with `t.Run`, `t.Helper()` in helpers, `t.Parallel()` where tests are independent, `t.TempDir()` and `t.Setenv()` for isolation.
- Use native fuzzing (`func FuzzParse(f *testing.F)`) for parsers and decoders; commit interesting corpus entries under `testdata/fuzz`.
- Keep benchmarks (`func BenchmarkX(b *testing.B)`) and compare with `benchstat`.

## Security

- Run `govulncheck` in CI; update the Go toolchain promptly for standard-library fixes.
- Validate and bound all external input; set timeouts on `http.Server` and clients; never build SQL or shell commands from strings.
- Use `crypto/rand` for tokens, never `math/rand`.

## Quick Checklist

- [ ] `go.mod` has a `go` version and `toolchain`; `go mod tidy` leaves no diff
- [ ] `gofmt`, `go vet`, `golangci-lint`, `go test -race` and `govulncheck` run in CI
- [ ] Errors wrapped with `%w`; context passed and honoured
- [ ] Goroutines have an owner and a shutdown path
