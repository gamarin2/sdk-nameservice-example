# SetName

## `Msg`

The naming convention for SDK `Msgs` is `Msg{{ .Action }}`. The first action to implement is `SetName`, a `Msg` that allows owner of an address to set the result of resolving a name. Start by defining `MsgSetName` in a new file called `./x/nameservice/msgs.go`:

```go
// MsgSetName defines a SetName message
type MsgSetName struct {
	NameID string
	Value  string
	Owner  sdk.AccAddress
}

// NewSetNameMsg is a constructor function for MsgSetName
func NewMsgSetName(name string, value string, owner sdk.AccAddress) MsgSetName {
	return MsgSetName{
		NameID: name,
		Value:  value,
		Owner:  owner,
	}
}
```

The `MsgSetName` has the three attributes needed to set the value for a name:
- `name` - The name trying to be set
- `value` - What the name resolves to
- `owner` - The owner of that name

> _*Note*_: the field name is `NameID` rather than `Name` as `.Name()` is the name of a method on the `Msg` interface.  This will be resolved in a [future update of the SDK](https://github.com/cosmos/cosmos-sdk/issues/2456).

Next, implement the `Msg` interface:

```go
// Type should return the name of the module
func (msg MsgSetName) Type() string { return "nameservice" }

// Name should return the action
func (msg MsgSetName) Name() string { return "set_name"}
```

These functions are used by the SDK to route `Msgs` to the proper module for handling. They also add human readable names to tags.

```go
// ValdateBasic Implements Msg.
func (msg MsgSetName) ValidateBasic() sdk.Error {
	if msg.Owner.Empty() {
		return sdk.ErrInvalidAddress(msg.Owner.String())
	}
	if len(msg.NameID) == 0 || len(msg.Value) == 0 {
		return sdk.ErrUnknownRequest("Name and/or Value cannot be empty")
	}
	return nil
}
```

`ValidateBasic` is used to provide some basic *stateless* checks on the validity of the msg.  In this case, we check that none of the attributes are empty. Note the use of the `sdk.Err*` types here. The SDK provides a set of error types that are frequently encountered by application developers.

```go
// GetSignBytes Implements Msg.
func (msg MsgSetName) GetSignBytes() []byte {
	b, err := json.Marshal(msg)
	if err != nil {
		panic(err)
	}
	return sdk.MustSortJSON(b)
}
```

`GetSignBytes` defines how the `Msg` gets encoded for signing.  In most cases this means marshal to sorted JSON. The output should not be modified.

```go
// GetSigners Implements Msg.
func (msg MsgSetName) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{msg.Owner}
}
```

`GetSigners` defines whose signature is required on a `Tx` in order for it to be valid.  In this case, for example, the `MsgSetName` requires that the `Owner` sign the transaction trying to reset what the name points to.

## `Handler`

Now that `MsgSetName` is specified, the next step is to define what action(s) needs to be taken when this message is received. This is the role of the `handler`. 

In a new file (`./x/nameservice/handler.go`) start with the following code:

```go
package nameservice

import (
	"fmt"
	"reflect"

	sdk "github.com/cosmos/cosmos-sdk/types"
)

// NewHandler returns a handler for "nameservice" type messages.
func NewHandler(keeper Keeper) sdk.Handler {
	return func(ctx sdk.Context, msg sdk.Msg) sdk.Result {
		switch msg := msg.(type) {
		case MsgSetName:
			return handleMsgSetName(ctx, keeper, msg)
		default:
			errMsg := fmt.Sprintf("Unrecognized nameservice Msg type: %v", reflect.TypeOf(msg).Name())
			return sdk.ErrUnknownRequest(errMsg).Result()
		}
	}
}
```

`NewHandler` is essentially a sub-router that directs messages coming into this module to the proper handler for the message. At the moment, there is only one `Msg`/`Handler`.

Now, you need to define the actual logic for handling the `MsgSetName` message in `handleMsgSetName`.:

> _*NOTE*_: The naming convention for handler names in the SDK is `handleMsg{{ .Action }}`

```go
// Handle MsgSetName
func handleMsgSetName(ctx sdk.Context, keeper Keeper, msg MsgSetName) sdk.Result {
	if !msg.Owner.Equals(keeper.GetOwner(ctx, msg.NameID)) { // Checks if the the msg sender is the same as the current owner
		return sdk.ErrUnauthorized("Incorrect Owner").Result() // If not, throw an error
	}
	keeper.SetName(ctx, msg.NameID, msg.Value) // If so, set the name to the value specified in the msg.
	return sdk.Result{}                      // return
}
```

In this function, check to see if the `Msg` sender is actually the owner of the name (`keeper.GetOwner`).  If so, they can set the name by calling the function on the `Keeper`.  If not, throw an error and return that to the user.

### Great, now owners can `SetName`s! But what if a name doesn't have an owner yet? Your module needs a way for users to buy names! Let us define [define the `BuyName` message](./buy-name.md).