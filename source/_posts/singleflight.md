---
layout: tile
title: Say Goodbye to Redundant Requests ✈ Golang Singleflight
date: 2023-07-07 14:00:27
tags: Go, Singleflight, Github
---

![](/images/singleflight/main.jpg)

Dalam pengembangan aplikasi, interaksi dengan `resource eksternal` dan `operasi I/O` seperti `API request`, `akses ke database`, dan `operasi pada filesystem` merupakan hal yang umum. Namun, seringkali kita dihadapkan pada masalah permintaan duplikat yang terjadi secara bersamaan ke resource tersebut. Hal ini dapat mengakibatkan penggunaan resource yang tidak efisien, peningkatan latensi, dan menghambat kinerja aplikasi.

Untungnya, ada sebuah pola desain yang disebut `Singleflight` yang efisien dalam mengatasi masalah permintaan duplikat ketika API request ke resource eksternal, I/O seperti akses ke database, filesystem, dan lain-lain. Pola Singleflight memungkinkan kita untuk menghindari duplikasi permintaan yang sama, sehingga mengoptimalkan penggunaan resource, mengurangi latensi, dan dapat meningkatkan kinerja aplikasi.

Sekarang kita akan menjelajahi konsep dan implementasi pola `Singleflight` yang diterapkan dalam bahasa pemrograman `Go`. Nantinya kita akan bermain dengan beberapa Studi kasus yang terkait penggunaan `Singleflight` untuk menangani permintaan duplikat saat melakukan panggilan API ke resource eksternal.

## Daftar Isi

-   [Apa itu Singleflight?]()
-   [Studi kasus Mendapatkan data github topic dari github API]()
    -   [Implementasi]()
    -   [Testing]()
-   [Mengulik implementasi internal `Singleflight` di Golang]()
-   [Kesimpulan]()
-   [Referensi]()

## Apa itu Singleflight?

`Singleflight` adalah pola desain yang digunakan untuk menghindari permintaan duplikat yang tidak perlu ke resource eksternal dan operasi I/O seperti akses ke database, filesystem, dan lain-lain. Pola ini memastikan bahwa hanya ada satu pemanggilan sebenarnya yang dilakukan untuk permintaan yang sama, sementara pemanggilan lainnya menunggu hasilnya. Hal ini membantu mengoptimalkan penggunaan resource, mengurangi latensi, dan meningkatkan kinerja aplikasi.

Konsep utama di balik `Singleflight` adalah koordinasi pemanggilan yang sama menggunakan mekanisme seperti `mutex` atau `channel`. Ketika sebuah permintaan datang, `Singleflight` memeriksa apakah permintaan tersebut sudah dilakukan sebelumnya. Jika sudah, pemanggilan baru akan menunggu hasil dari pemanggilan sebelumnya. Jika belum ada pemanggilan sebelumnya, pemanggilan baru akan dilakukan dan hasilnya disimpan untuk digunakan oleh pemanggilan berikutnya.

Pola Singleflight sangat berguna dalam situasi di mana terdapat banyak komponen dalam sistem yang memicu permintaan yang sama secara bersamaan, seperti dalam panggilan API, akses database, atau operasi pada filesystem.

Bahasa pemrograman `Go` sendiri telah menyediakan implementasi `Singleflight` yang dapat digunakan secara langsung melalui library `golang.org/x/sync/singleflight`. Selain itu, kita juga dapat membuat implementasi `Singleflight` sendiri dengan mudah menggunakan `mutex` atau `channel`. Tapi buat apa repot-repot bikin sendiri kalau sudah ada yang jadi, kan? 😁 Kendati demikian, nantinya kita tetap akan akan melihat bagaimana implementasi `Singleflight` di Golang bekerja di balik layar.

## Studi kasus Mendapatkan data github topic dari github API

Setelah mengenal konsep `Singleflight` selanjutnya kita akan bermain dengan studi kasus, kita akan melihat bagaimana `Singleflight` dapat digunakan untuk mengatasi masalah permintaan redundan ketika mengambil data github topic dari `GitHub API`. Kita akan menggunakan bahasa pemrograman `Go` dan library `github.com/google/go-github` untuk berinteraksi dengan Github API.

## Talk is cheap. Show me the code! 😁

Baiklah, berikut merupakan implementasi untuk mengambil data github topic dari GitHub API menggunakan `Singleflight`

`github_topic.go`

```go
package main

import (
	"context"
	"os"
	"sync"

	"github.com/google/go-github/v53/github"
	"golang.org/x/exp/slog"
	"golang.org/x/oauth2"
	"golang.org/x/sync/singleflight"
)

var (
	group           singleflight.Group
	githubClient    *github.Client
	logger          *slog.Logger
	requestsCounter sync.Map
)

func IncrementRequestCounter(key string) {
	v, _ := requestsCounter.LoadOrStore(key, 0)
	requestsCounter.Store(key, v.(int)+1)
}

func NewLogger(level slog.Level) {
	handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: level,
	})

	logger = slog.New(handler)
	slog.SetDefault(logger)
}

func NewGithubClient(accessToken string) {
	ts := oauth2.StaticTokenSource(
		&oauth2.Token{AccessToken: accessToken},
	)

	tc := oauth2.NewClient(context.Background(), ts)
	githubClient = github.NewClient(tc)
}

type GithubTopic struct {
	Result []*github.TopicResult
	Total  int
}

func GetGitHubTopicWithSingleflight(topic string) (*GithubTopic, error) {
	key := "github_topic_" + topic

	result, err, shared := group.Do(key, func() (interface{}, error) {
		logger.Info("Fetching data from GitHub API", slog.String("key", key))

		IncrementRequestCounter(key)

		topicSearchResult, _, err := githubClient.Search.Topics(context.Background(), topic, nil)
		if err != nil {
			return nil, err
		}

		return &GithubTopic{
			Result: topicSearchResult.Topics,
			Total:  topicSearchResult.GetTotal(),
		}, nil
	})
	if err != nil {
		return nil, err
	}

	if shared {
		logger.Info("Data is shared with others", slog.String("key", key))
	}

	return result.(*GithubTopic), nil
}

func GetGitHubTopic(topic string) (*GithubTopic, error) {
	logger.Info("Fetching data from GitHub API", slog.String("topic", topic))

	key := "github_topic_" + topic
	IncrementRequestCounter(key)

	topicSearchResult, _, err := githubClient.Search.Topics(context.Background(), topic, nil)
	if err != nil {
		return nil, err
	}

	return &GithubTopic{
		Result: topicSearchResult.Topics,
		Total:  topicSearchResult.GetTotal(),
	}, nil
}
```

Intinya, Fungsi `GetGitHubTopicWithSingleflight` menggunakan `Singleflight` yg diimplementasikan di Go untuk menghindari permintaan redundan ke github API. Pertama, kita membuat sebuah `key` unik dengan format `"github_topic_" + topic`, contohnya `github_user_gRPC`. Key ini akan digunakan sebagai identifier untuk setiap request. Selanjutnya, kita menggunakan fungsi `group.Do` untuk menjalankan fungsi callback hanya jika tidak ada goroutine lain yang sedang menjalankan permintaan dengan `key` yang sama.

> Mungkin setelah membaca paragraf diatas, kamu sudah bisa menebak kalau internal implementation `Singleflight` menggunakan struktur data `map` 😁

Ok Lanjut,

Dalam fungsi `callback`, kita melakukan penambahan total permintaan ke GitHub API menggunakan fungsi `IncrementRequestCounter` (nantinya ini dipakai untuk keperluan testing). Kemudian, kita menggunakan `githubClient.Search.Topics` untuk mendapatkan data topic dari github API.

Setelah permintaan selesai, hasilnya akan disimpan dalam variabel `result` yang dikembalikan oleh `group.Do`. Variable `shared` akan bernilai true sebagai flag jika ada goroutine lain yang sudah mengambil data sebelumnya.

Kita juga membuat fungsi `GetGitHubTopic` yang nantinya digunakan untuk membandingkan hasilnya dengan `GetGitHubTopicWithSingleflight`:

### Testing

Selanjutnya, kita akan membuat unit test untuk membandingkan hasil dari kedua fungsi tersebut:

`github_topic_test.go`

```go
package main

import (
	"sync"
	"testing"

	"golang.org/x/exp/slog"
)

func TestMain(m *testing.M) {
	accessToken := ""
	NewGithubClient(accessToken)
	NewLogger(slog.LevelInfo)

	requestsCounter = sync.Map{}

	m.Run()
}

func TestGetGitHubTopicWithSingleflight(t *testing.T) {
	topic := "gRPC"

	// Simulate multiple concurrent requestsCounter to the GitHub API
	concurrentRequests := 10
	var wg sync.WaitGroup
	topicResult := make(chan *GithubTopic, concurrentRequests)

	for i := 0; i < concurrentRequests; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			profile, err := GetGitHubTopicWithSingleflight(topic)
			if err != nil {
				t.Errorf("Error getting GitHub topic: %v", err)
				return
			}
			topicResult <- profile
		}()
	}

	wg.Wait()
	close(topicResult)

	v, ok := requestsCounter.Load("github_topic_" + topic)
	if !ok {
		t.Errorf("Error getting number of requestsCounter")
		return
	}

	if v.(int) != 1 {
		t.Errorf("Number of requestsCounter is not 1")
		return
	}
}

func TestGetGitHubTopic(t *testing.T) {
	topic := "gRPC"

	// Simulate multiple concurrent requestsCounter to the GitHub API
	concurrentRequests := 10
	var wg sync.WaitGroup
	topicResult := make(chan *GithubTopic, concurrentRequests)

	for i := 0; i < concurrentRequests; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			profile, err := GetGitHubTopic(topic)
			if err != nil {
				t.Errorf("Error getting GitHub topic: %v", err)
				return
			}
			topicResult <- profile
		}()
	}

	wg.Wait()
	close(topicResult)

	v, ok := requestsCounter.Load("github_topic_" + topic)
	if !ok {
		t.Errorf("Error getting number of requestsCounter")
		return
	}

	if v.(int) != concurrentRequests {
		t.Errorf("Number of requestsCounter is not %d", concurrentRequests)
		return
	}
}

```

Kita bisa lihat hasilnya pada gambar berikut, bahwa `TestGetGitHubTopicWithSingleflight` hanya melakukan 1 permintaan ke github API, sedangkan `TestGetGitHubTopic` melakukan 10 requests.

`TestGetGitHubTopicWithSingleflight`

![](/images/singleflight/test-1.png)

`TestGetGitHubTopic`

![](/images/singleflight/test-2.png)

## Mengulik implementasi internal Singleflight di Golang

Yosha, sekarang kita akan menelisir bagaimana `Singleflight` diimplementasikan di Golang.

Source code lengkapnya bisa dilihat di [sini](https://cs.opensource.google/go/x/sync/+/master:singleflight/singleflight.go). Kita akan fokus pada core logicnya saja.

-   Group

    ```go
    type Group struct {
    	mu sync.Mutex       // protects m
    	m  map[string]*call // lazily initialized
    }
    ```

    `Group` merupakan `struct` yang digunakan untuk membuat instance `Singleflight`. Group memiliki 2 field yaitu `mu` dan `m`. `mu` merupakan `mutex` yang digunakan untuk mengamankan `m` pada proses `concurrency`, dan `m` merupakan map yang menyimpan `key` dan `call` yang sedang berjalan.

-   Call

    ```go
    type call struct {
    	wg sync.WaitGroup

    	val interface{}
    	err error

    	dups  int
    	chans []chan<- Result
    }
    ```

    `Call` merupakan `struct` yang digunakan untuk menyimpan hasil dari `callback` yang dijalankan oleh `Do`. `wg` merupakan `WaitGroup` yang digunakan untuk menunggu goroutine selesai. `val` merupakan hasil dari `callback` yang dijalankan oleh `Do`. `err` merupakan error yang dihasilkan oleh `callback` yang dijalankan oleh `Do`. `dups` merupakan jumlah request yang menunggu. `chans` merupakan channel yang digunakan untuk mengirimkan hasil dari `callback` yang khusus untuk method `DoChan`.

-   Result

    ```go
    type Result struct {
    	Val    interface{}
    	Err    error
    	Shared bool
    }
    ```

    `Result` merupakan `struct` yang digunakan untuk mengembalikan hasil dari method `DoChan()`

-   Do

    ```go
    func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
    	g.mu.Lock()
    	if g.m == nil {
    		g.m = make(map[string]*call)
    	}
    	if c, ok := g.m[key]; ok {
    		c.dups++
    		g.mu.Unlock()
    		c.wg.Wait()

    		if e, ok := c.err.(*panicError); ok {
    			panic(e)
    		} else if c.err == errGoexit {
    			runtime.Goexit()
    		}
    		return c.val, c.err, true
    	}
    	c := new(call)
    	c.wg.Add(1)
    	g.m[key] = c
    	g.mu.Unlock()

    	g.doCall(c, key, fn)
    	return c.val, c.err, c.dups > 0
    }
    ```

    Method `Do` digunakan untuk menjalankan fungsi yang terkait dengan key tertentu dalam `Group`. method ini memastikan bahwa hanya satu eksekusi yang berjalan pada satu waktu untuk kunci yang sama. Jika ada panggilan duplikat, pemanggil duplikat akan menunggu hingga panggilan asli selesai dan menerima hasil yang sama.

    Okei, Mari kita bahas bagian-bagian dari method `Do` ini.

    ```go
    	if g.m == nil {
    		g.m = make(map[string]*call)
    	}
    ```

    Pada bagian ini, dilakukan pengecekan apakah `g.m` sudah diinisialisasi atau belum. Jika belum, maka kita akan menginisialisasi `g.m` dengan `map` kosong. (lazy initialization)

    ```go
    if c, ok := g.m[key]; ok {
    	c.dups++
    	g.mu.Unlock()
    	c.wg.Wait()

    	if e, ok := c.err.(*panicError); ok {
    		panic(e)
    	} else if c.err == errGoexit {
    		runtime.Goexit()
    	}
    	return c.val, c.err, true
    }
    ```

    Pada bagian ini, dilakukan pengecekan apakah `key` sudah ada di dalam `g.m` atau belum. Jika sudah ada, maka akan mengambil `call` yang terkait dengan `key` tersebut. Jika belum ada, maka nantinya akan dibuat instance `call` yang baru

    ketika `key` sudah ada di dalam `g.m`, maka `dups` akan di increment dengan 1. `dups` merupakan `total requests dengan key yang sama` yang menunggu hasil dari `callback` yang dijalankan oleh `Do`. Setelah itu, kita akan melakukan `unlock` pada `mutex` dan menunggu pemrosesan sebelumnya selesai selesai dengan `c.wg.Wait()`.

    ```go
    	c := new(call)
    	c.wg.Add(1)
    	g.m[key] = c
    	g.mu.Unlock()

    	g.doCall(c, key, fn)
    	return c.val, c.err, c.dups > 0
    ```

    Ketika key belum ada di dalam `g.m`, maka akan dibuat instance `call` yang baru, menambahkan `wg` dengan 1, dan menyimpan `call` tersebut ke dalam `g.m`. Setelah itu, dilakukan `unlock` / release pada `mutex` dan menjalankan `doCall` untuk menjalankan `callback` yang dijalankan oleh `Do`.

-   DoChan

    ```go
    if c, ok := g.m[key]; ok {
    	c.dups++
    	c.chans = append(c.chans, ch)
    	g.mu.Unlock()
    	return ch
    }
    c := &call{chans: []chan<- Result{ch}}
    c.wg.Add(1)
    g.m[key] = c
    g.mu.Unlock()

    go g.doCall(c, key, fn)
    ```

    Implementasi method `DoChan` juga mirip dengan `Do`, bedanya kalau `DoChan` akan mengembalikan channel yang digunakan untuk mengirimkan hasil dari `callback`, dan callback akan dijalankan di `goroutine` yang berbeda.

    Method `DoChan` ini cocok digunakan dalam kasus di mana si caller/pemanggil ingin menerima hasil asinkron dari pemanggilan fungsi di `Group`. Dengan menggunakan `DoChan`, pemanggil dapat mendapatkan hasil melalui `channel` ketika hasil tersebut sudah tersedia, tanpa harus secara aktif menunggu pemanggilan selesai. Jadi, pemanggil dapat melakukan hal lain ketika menunggu pemanggilan selesai.

-   doCall

    ```go
    	c.val, c.err = fn() ①

    	if g.m[key] == c { ②
    		delete(g.m, key)
    	}

    	for _, ch := range c.chans { ③
    		ch <- Result{c.val, c.err, c.dups > 0}
    	}
    ```

    Kode diatas hanya sebagian kecil dari implementasi doCall, karena menurut saya ini merupakan bagian yang perlu untuk di highlight. Pada bagian ①, dilakukan pemanggilan `callback` yang dijalankan oleh `Do` atau `DoChan`. Pada bagian ②, key yang telah selesai dijalankan akan dihapus dari `g.m` supaya next request dengan key yang sama akan membuat instance `call` yang baru (invalidate cache). Pada bagian ③, dilakukan pengiriman hasil dari `callback` ke `channel` yang digunakan oleh `DoChan`.

Mmm pusying? Enggak dong ya, ini saya buatkan flow ketika method `Do` dipanggil:

① lock mutex
② cek apakah key sudah ada di dalam g.m
③ jika sudah ada, maka tunggu sampai goroutine selesai (c.wg.Wait)
④ jika belum ada, maka buat instance call baru
⑤ unlock mutex
⑥ jalankan callback
⑦ jika callback selesai, val dan err akan terisi, jalankan c.wg.Done menandakan fungsi callback sudah selesai
⑦ hapus key dari g.m (cache invalidation)
⑧ kirim hasil ke channel yang digunakan oleh DoChan (kalau menggunakan DoChan)

## Kesimpulan

Jadi kesimpulannya, ya anda pasti sudah tau lah ya. Silahkan gunakan `singleflight` jika anda ingin menghindari `thundering herd` ketika melakukan request ke service lain. Tapi kita harus bijak kapan dan usecase apa saja yang cocok untuk menggunakan `singleflight`.

Terima kasih sudah membaca, semoga bermanfaat.

## Referensi

-   [Slog](https://thedevelopercafe.com/articles/logging-in-go-with-slog-a7bb489755c2)
-   [Search Topics](https://docs.github.com/en/rest/search/search?apiVersion=2022-11-28#search-topics)
-   [Generate dummy data](https://www.mockaroo.com/)
-   [Golang library for accessing the GitHub v3 API](https://github.com/google/go-github)
-   [Golang singleflight source code](https://cs.opensource.google/go/x/sync/+/master:singleflight/singleflight.go)
