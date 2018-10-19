# SetName

## `Msg`

The naming convention for SDK `Msgs` is `Msg{{ .Action }}`. The first action to implement is `SetName` so start by defining `MsgSetName` in a new file called `./x/nameservice/msgs.go` in the `nameservice` package, a `Msg` that allows owner of an address to set the result of resolving a name.

```go
// MsgSetName defines a SetName message
type MsgSetName struct {
	NameID string
	Value  string
	Owner  sdk.AccAddress
}

// NewSetNameMsg is a constructor function for MsgSetName
func NewMsgSetName(name string, value string, owner sdk.AccAddress) MsgSetName {
	return MsgBuyName{
		NameID: name,
		Value:  value,
		Owner:  owner,
	}
}
```

The `MsgSetName` has three attributes:
- `name` - The name trying to be set
- `value` - What the name resolves to
- `owner` - The owner of that name

Note the field name is `NameID` rather than `Name` as `.Name()` is the name of a method on the `Msg` interface.  This will be resolved in a [future update of the SDK](https://github.com/cosmos/cosmos-sdk/issues/2456).

```go
// Type should return the name of the module
func (msg MsgSetName) Type() string { return "nameservice" }

// Name should return the action
func (msg MsgSetName) Name() string { return "set_name"}
```

These functions are used by the SDK to route `Msgs` to the proper module for handling. They also add human readable names to tags. 

```go
// Implements Msg.
func (msg MsgSetName) ValidateBasic() sdk.Error {
	if msg.Owner.Empty() {
		return sdk.ErrInvalidAddress(msg.Owner.String())
	}
	if len(msg.NameID) == 0 || len(msg.Value) == 0 {
		return sdk.ErrUnknownRequest("Name and Value cannot be empty")
	}
	return nil
}
```
This is used to provide some basic *stateless* checks on the validity of the msg.  In this case, we check that none of the attributes are empty.

```go
// Implements Msg.
func (msg MsgSetName) GetSignBytes() []byte {
	b, err := json.Marshal(msg)
	if err != nil {
		panic(err)
	}
	return sdk.MustSortJSON(b)
}
```
This defines how the Msg gets encoded for signing.  This should usually be in JSON and should not be modified in most cases.

```go
// Implements Msg.
func (msg MsgSetName) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{msg.Owner}
}
```
This allows the Msg to define who's signature is required on a Tx in order for it to be valid.  In this case, for example, the `MsgSetName` requires that the `Owner` sign the transaction trying to reset what the name points to.

#### Handler

Now that we have the `MsgSetName` defined, we now have to define the handler that actually executes the Msg.

In a new file called `handler.go` in the `nameservice` package, we start off with:

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

This is essentially a subrouter that directs messages coming into this module to the proper handler for the message.  At the moment, we only have one Msg/Handler.

In the same file, we define the function `handleMsgSetName`.

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
In this function we check to see if the Msg sender is actually the owner of the name (which we get using `keeper.GetOwner`).  If so, we let them set the name by calling the function on the keeper.  If not, we throw an error.
