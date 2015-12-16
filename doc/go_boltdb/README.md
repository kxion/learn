[*back to contents*](https://github.com/gyuho/learn#contents)<br>

# Go: boltdb

- [Reference](#reference)
- [Write random data](#write-random-data)
- [Read all data](#read-all-data)
- [Write, read](#write-read)

[↑ top](#go-boltdb)
<br><br><br><br><hr>


#### Reference

- [`boltdb/bolt`](https://github.com/boltdb/bolt)

[↑ top](#go-boltdb)
<br><br><br><br><hr>


#### Write random data

```go
package main

import (
	"fmt"
	"math/rand"
	"os/user"
	"path/filepath"
	"time"

	"github.com/boltdb/bolt"
)

var (
	dbPath     = "test.db"
	bucketName = "test_bucket" 
	// 5GB
	// numKeys = 1250000

	// 2GB
	numKeys = 500000
	keyLen  = 100
	valLen  = 750
)

func init() {
	usr, err := user.Current()
	if err != nil {
		panic(err)
	}
	dbPath = filepath.Join(usr.HomeDir, dbPath)
}

func main() {
	fmt.Println("dbPath:", dbPath)
	fmt.Println("bucketName:", bucketName)

	// Open the dbPath data file in your current directory.
	// It will be created if it doesn't exist.
	db, err := bolt.Open(dbPath, 0600, &bolt.Options{Timeout: 1 * time.Second})
	if err != nil {
		panic(err)
	}
	defer db.Close()

	if err := db.Update(func(tx *bolt.Tx) error {
		b, err := tx.CreateBucket([]byte(bucketName))
		if err != nil {
			return err
		}
		for i := 0; i < numKeys; i++ {
			fmt.Println("Writing", i, "/", numKeys)
			if err := b.Put(randBytes(keyLen), randBytes(valLen)); err != nil {
				return err
			}
		}
		return nil
	}); err != nil {
		panic(err)
	}

	fmt.Println("Done! Saved:", db.Path())
}

func randBytes(n int) []byte {
	const (
		letterBytes   = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
		letterIdxBits = 6                    // 6 bits to represent a letter index
		letterIdxMask = 1<<letterIdxBits - 1 // All 1-bits, as many as letterIdxBits
		letterIdxMax  = 63 / letterIdxBits   // # of letter indices fitting in 63 bits
	)
	src := rand.NewSource(time.Now().UnixNano())
	b := make([]byte, n)
	for i, cache, remain := n-1, src.Int63(), letterIdxMax; i >= 0; {
		if remain == 0 {
			cache, remain = src.Int63(), letterIdxMax
		}
		if idx := int(cache & letterIdxMask); idx < len(letterBytes) {
			b[i] = letterBytes[idx]
			i--
		}
		cache >>= letterIdxBits
		remain--
	}
	return b
}

```

[↑ top](#go-boltdb)
<br><br><br><br><hr>


#### Read all data

```go
package main

import (
	"flag"
	"fmt"
	"os/user"
	"path/filepath"
	"syscall"
	"time"

	"github.com/boltdb/bolt"
)

/*
read with MAP_POPULATE flag...
bolt.Open took 2.879063477s
bolt read took: 51.952703ms

read without MAP_POPULATE flag...
bolt.Open took 1.568715ms
bolt read took: 13.795348869s
*/

var (
	dbPath     = "test.db"
	bucketName = "test_bucket"
	mapPop     = true
	writable   = false
)

func init() {
	usr, err := user.Current()
	if err != nil {
		panic(err)
	}
	dbPath = filepath.Join(usr.HomeDir, "test.db")

	mapPt := flag.Bool(
		"populate",
		true,
		"'true' for MAP_POPULATE flag.",
	)
	flag.Parse()
	mapPop = *mapPt
}

func main() {
	read(dbPath, mapPop)
}

func read(dbPath string, mapPop bool) {
	opt := &bolt.Options{Timeout: 5 * time.Minute, ReadOnly: true}
	if mapPop {
		fmt.Println("read with MAP_POPULATE flag...")
		opt = &bolt.Options{Timeout: 5 * time.Minute, ReadOnly: true, MmapFlags: syscall.MAP_POPULATE}
	} else {
		fmt.Println("read without MAP_POPULATE flag...")
	}

	to := time.Now()
	db, err := bolt.Open(dbPath, 0600, opt)
	if err != nil {
		panic(err)
	}
	defer db.Close()
	fmt.Println("bolt.Open took", time.Since(to))

	tr := time.Now()
	tx, err := db.Begin(writable)
	if err != nil {
		panic(err)
	}
	defer tx.Rollback()

	bk := tx.Bucket([]byte(bucketName))
	c := bk.Cursor()

	for k, v := c.First(); k != nil; k, v = c.Next() {
		// fmt.Printf("%s ---> %s.\n", k, v)
		_ = k
		_ = v
	}
	fmt.Println("bolt read took:", time.Since(tr))
}

```

[↑ top](#go-boltdb)
<br><br><br><br><hr>


#### Write, read

```go
package main

import (
	"fmt"
	"math/rand"
	"os"
	"time"

	"github.com/boltdb/bolt"
	"github.com/renstrom/shortuuid"
)

var (
	dbPath     = shortuuid.New() + ".db"
	bucketName = shortuuid.New()

	numKeys = 10
	keyLen  = 3
	valLen  = 7

	keys = make([][]byte, numKeys)
	vals = make([][]byte, numKeys)
)

func init() {
	fmt.Println("Generating random data...")
	for i := range keys {
		keys[i] = randBytes(keyLen)
		vals[i] = randBytes(valLen)
	}
	fmt.Println("Done with random data...")
}

func main() {
	fmt.Println("dbPath:", dbPath)
	fmt.Println("bucketName:", bucketName)

	defer os.Remove(dbPath)

	db, err := bolt.Open(dbPath, 0600, &bolt.Options{Timeout: 1 * time.Second})
	if err != nil {
		panic(err)
	}
	defer db.Close()

	if err := db.Update(func(tx *bolt.Tx) error {
		b, err := tx.CreateBucket([]byte(bucketName))
		if err != nil {
			return err
		}
		for i := range keys {
			if err := b.Put(keys[i], vals[i]); err != nil {
				return err
			}
		}
		return nil
	}); err != nil {
		panic(err)
	}

	if err := db.View(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte(bucketName))
		for i := range keys {
			fmt.Printf("%s ---> %s\n", keys[i], b.Get(keys[i]))
		}
		return nil
	}); err != nil {
		panic(err)
	}
	fmt.Println("Done with db.View")
}

/*
Generating random data...
Done with random data...
dbPath: xHUfNFaPy4YPcKDhbVC3qM.db
bucketName: gSjrPgznxEa8q7roSRXpLd
zWN ---> LKckYKJ
yLp ---> lvWgBBc
BAD ---> xasSjyf
ilB ---> wVWExop
sSZ ---> kSwzVtf
Ntv ---> NkcpxBO
EVq ---> dXnWnZR
PJu ---> TTQCqLc
WEU ---> HyCQkFw
dnL ---> WYBvMaH
Done with db.View
*/

func randBytes(n int) []byte {
	const (
		letterBytes   = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
		letterIdxBits = 6                    // 6 bits to represent a letter index
		letterIdxMask = 1<<letterIdxBits - 1 // All 1-bits, as many as letterIdxBits
		letterIdxMax  = 63 / letterIdxBits   // # of letter indices fitting in 63 bits
	)
	src := rand.NewSource(time.Now().UnixNano())
	b := make([]byte, n)
	for i, cache, remain := n-1, src.Int63(), letterIdxMax; i >= 0; {
		if remain == 0 {
			cache, remain = src.Int63(), letterIdxMax
		}
		if idx := int(cache & letterIdxMask); idx < len(letterBytes) {
			b[i] = letterBytes[idx]
			i--
		}
		cache >>= letterIdxBits
		remain--
	}
	return b
}

```

[↑ top](#go-boltdb)
<br><br><br><br><hr>
