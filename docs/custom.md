## Как сделать свою пушку

### На примере gRPC

Для стрельб по gRPC мы используем тул yandex/pandora
(https://github.com/yandex/pandora), пишем для него кастомную пушку.

Для стрельб по gRPC нужны будут протофайлы вашего сервиса и написать
новый тип пушки для тула, после чего пандора компилится в бинарник с
этой пушкой внутри.
Для начала возьмём шаблон кастомной пушки и перепишем для него метод
`shoot`

```
package main

import (
	"net/http"
	"strings"
	"strconv"
	"time"
	"github.com/spf13/afero"
	"github.com/yandex/pandora/cli"
	"github.com/yandex/pandora/components/phttp/import"
	"github.com/yandex/pandora/core"
	"github.com/yandex/pandora/core/aggregator/netsample"
	"github.com/yandex/pandora/core/import"
	"github.com/yandex/pandora/core/register"
	pb "path/to/protofile"
	"google.golang.org/grpc"
	"log"
)

type Ammo struct {
	Tag         string
	Param1      string
	Param2      string
	Param3      string
}

type Sample struct {
	URL              string
	ShootTimeSeconds float64
}

type GunConfig struct {
	Target string `validate:"required"` // Configuration will fail, without target defined
}

type Gun struct {
	// Configured on construction.
	client grpc.ClientConn
	conf   GunConfig
	// Configured on Bind, before shooting
	aggr core.Aggregator // May be your custom Aggregator.
	core.GunDeps
}

func NewGun(conf GunConfig) *Gun {
	return &Gun{conf: conf}
}

func (g *Gun) Bind(aggr core.Aggregator, deps core.GunDeps) error {
	conn, err := grpc.Dial(
		g.conf.Target,
                grpc.WithInsecure(),
                grpc.WithTimeout(time.Second),
                grpc.WithUserAgent("pandora load test"))
	if err != nil {
		log.Fatalf("FATAL: %s", err)
	}
	g.client = *conn
	g.aggr = aggr
	g.GunDeps = deps
	return nil
}

func (g *Gun) Shoot(ammo core.Ammo) {
	customAmmo := ammo.(*Ammo) // Shoot will panic on unexpected ammo type. Panic cancels shooting.
	g.shoot(customAmmo)
}

func (g *Gun) shoot(ammo *Ammo) {
	//start := time.Now()
	code := 0
	sample := netsample.Acquire(ammo.Tag)

	conn := g.client
	client := pb.NewOpinionServiceClient(&conn)

	// prepare list of ids from ammo
	var itemIDs []int64
	for _, id := range strings.Split(ammo.Param1, ",") {
		if id == "" {
			continue
		}
		itemID, err := strconv.ParseInt(id, 10, 64)
		if err != nil {
			log.Printf("ERROR: %s", err)
			return
		}

		itemIDs = append(itemIDs, itemID)
	}

	// make scores request
	req := http.Request{}
	out, err := client.GetProductScores(req.Context(), &pb.GetProductScoresRequest{ItemIds: itemIDs})

	if err != nil {
		code = 0
	}

	if out != nil {
		code = 200
	}

	defer func() {
		sample.SetProtoCode(code)
		g.aggr.Report(sample)
	}()
}

func main() {
	//debug.SetGCPercent(-1)
	// Standard imports.
	fs := afero.NewOsFs()
	coreimport.Import(fs)
	// May not be imported, if you don't need http guns and etc.
	phttp.Import(fs)

	// Custom imports. Integrate your custom types into configuration system.
	coreimport.RegisterCustomJSONProvider("custom_provider", func() core.Ammo { return &Ammo{} })

	register.Gun("gun_product-review-api-GetProductScores", NewGun, func() GunConfig {
		return GunConfig{
			Target: "default target",
		}
	})

	cli.Run()
}
```

В скрипте выше мы описали логику работы со своим сервисом в методе
`shoot` (всё остальное трогать не нужно), в func main() назвали только
что сделанную пушку `gun_product-review-api-GetProductScores`. Теперь
компилим получившийся код пушки (go build my_custom_gun.go) и
получившийся бинарник - это наша стрелялка с пушкой внутри.

Теперь делаем конфиг для пандоры, в котором указываем имя кастомной
пушки и запускаем бинарник с этим конфигом. В случае стрельбы через
yandex-tank, в pandora_cmd нужно будет указать абсолютный путь к нашему
скомпиленному только что бинарнику.

```
pools:
  - id: HTTP pool
    gun:
      type: gun_product-review-api-GetProductScores
      target: "your_grpc_host:your_grpc_port"
    ammo:
      type: custom_provider
      source:
        type: file
        path: ./json.ammo
    result:
      type: phout
      destination: ./phout.log
    rps: {duration: 30s, type: line,  from: 1, to: 2}
    startup:
      type: once
      times: 10
```

Формат патронов (тестовые данные, передающиеся построчно в пушку,
1 строка = 1 запрос). Эти данные доступны у инстанса класса ammo,
передаваемого в метод shoot как атрибуты класса.
```
{"tag": "/MyCase1", "Param1": "146837693,146837692,146837691"}
{"tag": "/MyCase2", "Param2": "555", "Param1": "500002"}
```

При стрельбе кастомной пушкой не стоит забывать самостоятельно
проставлять коды ответов (sample.SetProtoCode(code))

