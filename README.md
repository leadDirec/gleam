# gleam
a Go based distributed execution system. The computation can be written in Lua/Luajit or Unix Pipe tools.

# Installation
1. Install Luajit
2. Put this customized MessagePack.lua under a folder where luajit can find it.
```
  https://github.com/chrislusf/gleam/blob/master/examples/tests/MessagePack.lua
  sudo cp MessagePack.lua /usr/local/share/luajit-2.0.4/
```

# Documentation
* [Gleam Wiki] (https://github.com/chrislusf/gleam/wiki)
* [Gleam Flow API GoDoc](https://godoc.org/github.com/chrislusf/gleam/flow)

# Standalone Example

The full source code, not snippet, for word count:
```
package main

import (
	"os"

	"github.com/chrislusf/gleam"
)

func main() {

	gleam.New().TextFile("/etc/passwd").FlatMap(`
		function(line)
			return line:gmatch("%w+")
		end
	`).Map(`
		function(word)
			return word, 1
		end
	`).Reduce(`
		function(x, y)
			return x + y
		end
	`).SaveTextTo(os.Stdout, "%s,%d")
}

```

Another way to do the similar:
```
package main

import (
	"os"

	"github.com/chrislusf/gleam"
)

func main() {

	gleam.New().TextFile("/etc/passwd").FlatMap(`
		function(line)
			return line:gmatch("%w+")
		end
	`).Pipe("sort").Pipe("uniq -c").SaveTextTo(os.Stdout, "%s")
}

```


## Parallel Execution
One limitation for unix pipes is that they are easy for one single pipe, but not easy to parallel.

With Gleam this becomes very easy. (And this can be in distributed mode too!)

This example get a list of file names, partitioned into 3 groups, and then process them in parallel.
This flow can be changed to use Pipe() also, of course.

```
// word_count.go
package main

import (
	"log"
	"os"
	"path/filepath"

	"github.com/chrislusf/gleam"
)

func main() {

	fileNames, err := filepath.Glob("/Users/chris/Downloads/txt/en/ep-08-*.txt")
	if err != nil {
		log.Fatal(err)
	}

	gleam.New().Lines(fileNames).Partition(3).PipeAsArgs("cat $1").FlatMap(`
      function(line)
        return line:gmatch("%w+")
      end
    `).Map(`
      function(word)
        return word, 1
      end
    `).Reduce(`
      function(x, y)
        return x + y
      end
    `).SaveTextTo(os.Stdout, "%s\t%d")

}

```

# Distributed Computing
## Setup Gleam Cluster
Start a gleam master and serveral gleam agents
```
// start "gleam master" on a server
> go get github.com/chrislusf/gleam/distributed/gleam
> gleam master --address=":45326"

// start up "gleam agent" on some diffent server or port
// if a different server, remember to install Luajit and copy the MessagePack.lua file also.
> gleam agent --dir=2 --port 45327 --host=127.0.0.1
> gleam agent --dir=3 --port 45328 --host=127.0.0.1
```

## Change Execution Mode.
From gleam.New(), change to gleam.NewDistributed(), or gleam.New(gleam.Distributed)
```
  // local mode
  gleam.New()
  gleam.New(gleam.Local)
  
  // distributed mode
  gleam.NewDistributed()
  gleam.New(gleam.Distributed)
```
gleam.New(gleam.Local) and gleam.New(gleam.Distributed) are provided to dynamically change the execution mode.
