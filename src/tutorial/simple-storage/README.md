# Simple Storage

So far the two examples we have looked at have explored slicing bytes from calldata, storing in memory and returning values. Now we're going to address the missing piece of the puzzle that all EVM devs fear, storage.

## Storage in Huff

Thankfully working with storage isn't too complicated, Huff abstracts keeping track of storage variables through the `FREE_STORAGE_POINTER()` keyword. An example of which will be shown below:

```
#define constant STORAGE_SLOT0 = FREE_STORAGE_POINTER()
#define constant STORAGE_SLOT1 = FREE_STORAGE_POINTER()
#define constant STORAGE_SLOT2 = FREE_STORAGE_POINTER()
```

Storage slots are simply keys in a very large array where contracts keep their state. The compiler will assign `STORAGE_SLOT0` the value `0`, `STORAGE_SLOT1` the value `1` etc. at compile time. Throughout your code you just reference the storage slots the same way constants are used in any language.

## Setting storage

First define the constant that will represent your storage slot using the `FREE_STORAGE_POINTER()` keyword.

```
#define constant VALUE = FREE_STORAGE_POINTER()
```

We can then reference this slot throughout the code by wrapping it in square brackets - like so `[VALUE]`. The example below demonstrates a macro that will store the value 5 in the slot [VALUE].

```
#define macro SET_5() = takes(0) returns(0) {
    0x5             // [0x5]
    [VALUE]         // [value_slot_pointer, 0x5]
    sstore          // []
}
```

Test this out interactively [here](https://www.evm.codes/playground?unit=Wei&codeType=Bytecode&code='6005600055'_) ([VALUE] has been hardcoded to 0)

## Reading from storage

Now you know how to write to storage, reading from storage is trivial. Simply replace `sstore` with `sload` and your ready to go. We are going to extend our example above to both write and read from storage.

```
#define macro SET_5_READ_5() = takes(0) returns(0) {
    0x5
    [VALUE]
    sstore

    [VALUE]
    sload
}
```

Nice! Once again you can test this out over at [evm.codes](https://www.evm.codes/playground?unit=Wei&codeType=Bytecode&code='6005600055600054'_). Notice how 5 reappears on the stack after executing the `sload` instruction.

## Simple Storage Implementation

Now we can read and write to storage, lets attempt the famous SimpleStorage starter contract from remix.

First off, lets create our interface:

```
#define function setValue(uint256) nonpayable returns ()
#define function getValue() nonpayable returns (uint256)
```

Now lets define our storage slots:

```
#define constant VALUE = FREE_STORAGE_POINTER()
```

Onto the fun part, the logic. Remember from the addTwo example we can read calldata in 32 byte chunks using the `calldataload` opcode, lets use that knowledge to get read our uint256.

```
#define macro SET_VALUE() = takes(0) returns(0) {
   // Read uint256 from calldata, here we are telling the EVM to start reading at byte 4, i.e. skip it and read from where that 4 bytes ends.

    0x04            // [0x04]
    calldataload    // [value]

    // Get pointer and store
    [VALUE]         // [value_ptr, value]
    sstore          // []
}
```

Now, what is going on with `0x04 calldataload` and why we're adding 0x04 there? In our previous setter examples on this page, we knew that we were going to set the value 5 (0x5), because we ourselves decided to that value. But in this function, we do not know in advance what the value will be. So we need to read it from calldata.

Remember, calldata is the input data sent to a smart contract when a function is called. Since we've defined the interface for setValue (`#define function setValue(uint256) nonpayable returns ()`) and that the argument is an uint256, we know that the argument will always be 32 bytes long. The catch is, the first 4 bytes of this calldata is reserved for the `function selector`. This function selector is the unique identifier that tells the EVM which function to call. But we do not need to store it, because we don't want it, we want the function argument, which comes AFTER that first 4 bytes that is reserved to the function selector. We want to work with the part that is after the first 4 bytes. That is why by writing `0x04` before calldataload we are telling the EVM that we want to skip the first 4 bytes, and read the next 32 bytes that is the argument given to the function. Had we not done that, we would've feed the EVM the wrong value.

After completing the previous examples we hope that writing Huff is all starting to click! This pattern will be extremely common when writing your own contracts, reading from calldata then storing values in memory or storage.

Next up is reading the stored value.

```
#define macro GET_VALUE() = takes(0) returns(0) {
    // Read uint256 from storage
    [VALUE]         // [value_ptr]
    sload           // [value]

    // Store the return value in memory
    0x00            // [0x00, value]
    mstore          // []

    // Return the first 32 bytes of memory containing our uint256
    //0x20 is the HEX representation of the number 32, i.e. 32 bytes
    0x20            // [0x20]
    0x00            // [0x00, 0x20]
    return          // []
}
```

Here, we are first loading the value from storage (from the VALUE storage slot) onto the stack. Now, the value is at the top of the stack. Then, with `0x00 mstore` we take the value from the stack and move into memory at address 0x00 (the first memory slot). Note that the stack is now empty again.

Now that we stored the value in memory, we want to return it. But before returning it, we need to specify how much data to return, and where does that data come from. We do that with `0x20 0x00 return`. 0x20 is the HEX representation of the number 32. So we are essentially telling to the EVM "Hey, return 32 bytes of data starting at address 0x00". To make it clear, we are returning 32 bytes, because that is how many bytes our uint256 argument is.

To call our new macros from external functions we have to create a dispatcher!

```
#define macro MAIN() = takes(0) returns(0) {

    // Get the function selector
    0x00 calldataload 0xe0 shr

    dup1 __FUNC_SIG(setValue) eq setValue jumpi // Compare function selector to setValue(uint256)
    dup1 __FUNC_SIG(getValue) eq getValue jumpi // Compare the function selector to getValue()

    // dispatch
    setValue:
        SET_VALUE()
    getValue:
        GET_VALUE()

    0x00 0x00 revert
}
```

Here, with the line `0x00 calldataload 0xe0 shr` we are getting the function selector of the function the macro will direct us to by doing the following: First, with `0x00 calldataload` we are loading the first 32 bytes of the calldata into the stack. As we know, the first 4 bytes of the calldata contain the function selector.

Then, `0xe0 shr` shifts the bytes 224 bits to the right. Why and how? 0xe0 is the HEX representation of 224. Our argument was int256, i.e. had 256 bits, that is 32 bytes. We shift what we have 224 bits (28 bytes) to right, so that we are left with the first 32 bits (4 bytes) that is the function selector.

It is a bit confusing at first, just remember that 1 byte is 8 bits, so 32 bytes is 256 bits, and we need the first 4 bytes (function selector) that is 32 bits. 256 - 32 = 224 (all in bits). This is why we want to get rid of the last 224 bits (28 bytes) so that are left with the first 32 bits (4 bytes) that is the function selector. Huff for real eh!

Then, we have the line `dup1 __FUNC_SIG(setValue) eq setValue jumpi ` which checks if the function selector is equal to the function signature of setValue(uint256). If it is, jump to the `setValue` macro. The code does that by `dup1` OPCODE, which tells the EVM to "duplicate the 1st stack item", compare it to the function selector of setValue, and if they are equal, jump to the `setValue` macro.

The next line ` dup1 __FUNC_SIG(getValue) eq getValue jumpi` does the same for getValue. `jumpi` OPCODE means "jump if equal".

Then, we have these instructions

```
setValue:
    SET_VALUE()
getValue:
    GET_VALUE()
```

These work as follows: setValue: and getValue are labels. Labels act as “markers” in the code, similar to a “goto” statement in other languages. When the jumpi instruction tells the EVM to jump to setValue or getValue, execution continues from the line where that label is defined.

SET_VALUE() and GET_VALUE() are macros that have been defined elsewhere in the Huff code. When the EVM reaches one of these labels, it will execute the corresponding macro. This is similar to calling a function in other languages.

IF the function selector doesn't match either setValue or getValue, execution reaches this line.
`0x00 0x00 revert` tells the EVM to revert the transaction with no data returned, signaling that the function was not found or not supported.

Now all of it together!

```
// Interface
#define function setValue(uint256) nonpayable returns ()
#define function getValue() nonpayable returns (uint256)

// Storage
#define constant VALUE = FREE_STORAGE_POINTER()

// External function macros

// setValue(uint256)
#define macro SET_VALUE() = takes(0) returns(0) {
    // Read uint256 from calldata, remember to read from byte 4 to allow for the function selector!
    0x04            // [0x04]
    calldataload    // [value]

    // Get pointer and store
    [VALUE]         // [value_ptr, value]
    sstore          // []
}

// getValue()
#define macro GET_VALUE() = takes(0) returns(0) {
    // Read uint256 from storage
    [VALUE]         // [value_ptr]
    sload           // [value]

    // Store the return value in memory
    0x00            // [0x00, value]
    mstore          // []

    // Return the first 32 bytes of memory containing our uint256
    0x20            // [0x20]
    0x00            // [0x00, 0x20]
    return          // []
}

// Main
#define macro MAIN() = takes(0) returns(0) {
    // Get the function selector
    0x00 calldataload 0xe0 shr

    dup1 __FUNC_SIG(setValue) eq setValue jumpi // Compare function selector to setValue(uint256)
    dup1 __FUNC_SIG(getValue) eq getValue jumpi // Compare the function selector to getValue()

    // dispatch
    setValue:
        SET_VALUE()
    getValue:
        GET_VALUE()

    0x00 0x00 revert
}
```

Congratulations! You've made it through the crust of writing contracts in Huff. For your next steps we recommend taking what you have learned so far in addTwo, "Hello, World!" and SimpleStorage into a testing framework like [Foundry](https://docs.huff.sh/tutorial/huff-testing/). Happy hacking!
