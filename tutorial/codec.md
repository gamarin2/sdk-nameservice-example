# Codec File

To [register your types with Amino](https://github.com/tendermint/go-amino#registering-types) so that they can be encoded/decoded by your module there is a bit of code that needs to be placed in `./x/nameservice/codec.go`:

```go
package nameservice

import (
	"github.com/cosmos/cosmos-sdk/codec"
)

// RegisterCodec registers concrete types on wire codec
func RegisterCodec(cdc *codec.Codec) {
	cdc.RegisterConcrete(MsgSetName{}, "nameservice/SetName", nil)
	cdc.RegisterConcrete(MsgBuyName{}, "nameservice/BuyName", nil)
}
```

### Next you need to define [CLI interactions](./cli.md) with your module.
