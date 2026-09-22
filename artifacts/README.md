# Distributor crash-loop — before/after evidence

Evidence for the two defects in `processAttributes`
(`modules/distributor/distributor.go`),
the distributor's per-span attribute pass.

Screenshots are for sharing;
the `.txt` files are the raw `go test` output they were rendered from.

## Screenshots

| File | Shows |
| --- | --- |
| `05-before-after-summary.png` | Headline before/after table — start here |
| `01-before-panic.png` | The crash: a span attribute with no value nil-dereferences at `distributor.go:875` |
| `02-before-truncation-aliases-original.png` | Truncated attributes still point into the original oversized allocation |
| `03-heap-retention-before-after.png` | 50 MiB of oversized attributes truncated to 128 bytes each: 52,485,768 B retained before, 162,632 B after |
| `04-after-tests-pass.png` | Package green, including `-race` |

## Raw output

| File | Command |
| --- | --- |
| `before-panic.txt` | `go test ./modules/distributor/ -run TestProcessAttributesWithoutValue -v` |
| `before-heap-retention.txt` | `go test ./modules/distributor/ -run TestProcessAttributesTruncationReleasesOriginal -v` |
| `before-heap-measurement.txt`, `after-heap-measurement.txt` | The heap harness below |
| `before-bench.txt`, `after-bench.txt` | `go test ./modules/distributor/ -run XXX -bench BenchmarkProcessAttributes -benchmem -count 3` |
| `after-tests-pass.txt` | `go test ./modules/distributor/ -run TestProcessAttributes -v` and `go test -race ./modules/distributor/...` |

## Benchmark summary

`BenchmarkProcessAttributes`, median of three runs.
Truncation now copies, so it costs one right-sized allocation per truncated
string — paid only when an attribute actually exceeds the limit.
The common path, where nothing is oversized, is unchanged.

| Case | Before | After |
| --- | --- | --- |
| `within_limit` | 1107 ns/op, 1440 B/op, 61 allocs/op | 1116 ns/op, 1440 B/op, 61 allocs/op |
| `value_truncated` | 1107 ns/op, 1440 B/op, 61 allocs/op | 5039 ns/op, 21920 B/op, 81 allocs/op |
| `key_and_value_truncated` | 1115 ns/op, 1440 B/op, 61 allocs/op | 8700 ns/op, 42400 B/op, 101 allocs/op |

## Reproducing the heap measurement

The retention numbers come from a throwaway harness rather than a committed test,
because asserting on `runtime.MemStats` in CI is flaky.
The committed `TestProcessAttributesTruncationReleasesOriginal` locks the
underlying invariant instead: a truncated string must not alias the original
backing array.

To re-measure, drop this in `modules/distributor/` and run
`go test ./modules/distributor/ -count=1 -run TestZZHeapMeasurement -v`:

```go
package distributor

import (
	"encoding/binary"
	"runtime"
	"strings"
	"testing"

	"github.com/stretchr/testify/require"

	v1_common "github.com/grafana/tempo/pkg/tempopb/common/v1"
	v1_resource "github.com/grafana/tempo/pkg/tempopb/resource/v1"
	v1 "github.com/grafana/tempo/pkg/tempopb/trace/v1"
)

const (
	probeSpans       = 200
	probeAttrSize    = 1 << 18 // 256 KiB per attribute
	probeMaxAttrSize = 128
)

func makeOversizedBatch() []*v1.ResourceSpans {
	batch := &v1.ResourceSpans{
		Resource:   &v1_resource.Resource{},
		ScopeSpans: []*v1.ScopeSpans{{}},
	}
	for i := 0; i < probeSpans; i++ {
		traceID := make([]byte, 16)
		spanID := make([]byte, 8)
		binary.BigEndian.PutUint64(traceID[8:], uint64(i+1))
		binary.BigEndian.PutUint64(spanID, uint64(i+1))
		batch.ScopeSpans[0].Spans = append(batch.ScopeSpans[0].Spans, &v1.Span{
			TraceId: traceID,
			SpanId:  spanID,
			Attributes: []*v1_common.KeyValue{{
				Key:   "big",
				Value: &v1_common.AnyValue{Value: &v1_common.AnyValue_StringValue{StringValue: strings.Repeat("x", probeAttrSize)}},
			}},
		})
	}
	return []*v1.ResourceSpans{batch}
}

// rebatchOversized builds the batch, truncates it, and returns only the rebatched
// traces. The input batch goes out of scope on return, so anything still live is
// reachable solely through the truncated spans.
func rebatchOversized(t *testing.T) []*rebatchedTrace {
	t.Helper()

	_, traces, truncated, _, err := requestsByTraceID(makeOversizedBatch(), "test", probeSpans, probeMaxAttrSize)
	require.NoError(t, err)
	require.Equal(t, probeSpans, truncated.Span)
	return traces
}

func TestZZHeapMeasurement(t *testing.T) {
	var baseline, retained runtime.MemStats

	runtime.GC()
	runtime.ReadMemStats(&baseline)

	traces := rebatchOversized(t)

	runtime.GC()
	runtime.GC()
	runtime.ReadMemStats(&retained)

	t.Logf("oversized attribute bytes pushed:        %10d", probeSpans*probeAttrSize)
	t.Logf("attribute bytes after truncation:        %10d", probeSpans*probeMaxAttrSize)
	t.Logf("live heap held by rebatched traces:      %10d", retained.HeapAlloc-baseline.HeapAlloc)

	runtime.KeepAlive(traces)
}
```
