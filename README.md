# workclient

workclient is a simple worker library for handling boilerplate things such
as pipelining, graceful shutdowns, attaching to signals, logging events,
sending heartbeat checks to etcd (when configured), sending internal
metrics to statsd/influx/graphite and general execution.

View the [docs](http://godoc.org/github.com/nyxtom/workclient).

## Installation

```
$ go get github.com/nyxtom/workclient
```

## Example

```go
import (
	"io"
	"net/http"
	"time"

	"github.com/nyxtom/workclient"
)

type SimpleService struct {
	workclient.WorkClient
	exit chan error
	ticker *time.Ticker
}

func NewSimpleService(config *workclient.Config) *SimpleService {
	service := new(SimpleService)
	service.Configure(config, service.execute, service.stopExecute)
	service.exit = make(chan error)
	service.ticker = time.NewTicker(time.Second)
	return service
}

func (s *SimpleService) execute() {
	for {
		select {
		case <-s.exit:
			s.ticker.Stop()
			return
		case <-s.ticker.C:
			fmt.Println("Running...")
		}
	}
}

func (s *SimpleService) stopExecute() {
	s.exit <-nil
}

func main() {
	cfg := new(workclient.Config)
	service := NewSimpleService(cfg)

	sc := make(chan os.Signal, 1)
	signal.Notify(sc,
		syscall.SIGHUP,
		syscall.SIGINT,
		syscall.SIGTERM,
		syscall.SIGQUIT,
		syscall.SIGKILL,
		os.Interrupt)

	go func() {
		<-sc
		service.Close()
	}()

	service.Run()
}
```

# LICENSE

MIT
