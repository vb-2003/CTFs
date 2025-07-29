# [WWCTF] - Writeup

## Challenge: Solidity Jail 1
**Category: Blockchain**  

---

### Description
*Challenge Creator: zeptoide

Bash Jail? Boring. Pyjail? Too Common.

Introducing for the first time, Solidity Jail!
Make a contract to read the flag!*

---

### Analysis
In this challenge, we are given a jailshell based in Solidity and we need to get the flag. We are given the code the jail shell is using:

```python
blacklist = [
    "flag",
    "transfer",
    "address",
    "this",
    "block",
    "tx",
    "origin",
    "gas",
    "fallback",
    "receive",
    "selfdestruct",
    "suicide"
]
if any(banned in body for banned in blacklist):
    raise ValueError(f"Blacklisted string found in contract.")


source = f"""// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Solution {{
    function main() external returns (string memory) {{
{body}
    }}
}}
"""

print("Final contract with inserted main() body:")
print(source)
```

This snippet from the code shows that there are blacklisted words that we cannot use, and that whatever we input is inserted into the main function.

We are also given the ABI of the contract:

```python
contr_abi = [{"inputs":[],"name":"flag","outputs":[{"internalType":"string","name":"","type":"string"}],"stateMutability":"view","type":"function"},{"inputs":[{"internalType":"bytes","name":"_bytecode","type":"bytes"},{"internalType":"bytes32","name":"_salt","type":"bytes32"}],"name":"run","outputs":[{"internalType":"bool","name":"success","type":"bool"},{"internalType":"bytes","name":"result","type":"bytes"}],"stateMutability":"nonpayable","type":"function"}]
```

This shows us what functions the contract has. We can see that it has a function called `flag()`, which takes 0 parameters and returns a string. In order to get the flag, we need to call `flag()`. However, flag is one of the blacklisted words. We need to call the `flag()` function without directly using the word flag.

---

### Approach
Solidity has a way for functions to be called by their function selectors, rather than the name of the function. In Solidity, a function selector is the first 4 bytes of the hashed function name and parameters. The syntax of this is:

```sol
bytes4(keccak256("functionName(parameterType1,parameterType2,etc)"))
```

We can use the `call` function with the selector. Note that `call` returns a boolean which indicates whether it was successful or not, as well as the output of the function called. Since we only care about the output, we can ignore the success variable using a blank tuple element by doing `(, out)`. To do this, we can do:
```sol
(, out) = msg.sender.call(abi.encodeWithSelector(FUNCTION_SELECTOR));
abi.decode(out, (string));
```
To do this, we need to get the function selector for `flag()`. To get a function signature, we can use foundry. The command is:
```bash
cast sig "functionName(paramType1,paramType2,etc)"
```
To get the signature of flag(), we can do:
```bash
cast sig "flag()"
```
This outputs `0x890eba68`, which is the function selector for `flag`.

---

### Exploitation / Solution
We can put the function selector we just got into our solution
```sol
(, out) = msg.sender.call(abi.encodeWithSelector(0x890eba68));
abi.decode(out, (string));
```
<img width="744" height="477" alt="Screenshot 2025-07-28 at 5 00 05 PM" src="https://github.com/user-attachments/assets/1fb4d6a7-9942-4a51-bf4c-77704b894aef" />

This gets us the flag: `wwf{y0u_4r3_7h3_7ru3_m4573r_0f_s0l1d17y}`

---

### Better Solution
In my original solution, I used `abi.encodeWithSelector(0x890eba68)`. However, this is not needed. `abi.encodeWithSelector` takes the parameters (functionSelector, param1, param2, etc) and encodes them. Since our function, `flag()`, does not take any parameters, `abi.encodeWithSelector(0x890eba68)` returns `0x890eba68`. When calling a function without parameters, the function selector can be used directly, without `abi.encodeWithSelector`. We could have just used:
```sol
(, out) = msg.sender.call(hex'890eba68');
abi.decode(out, (string));
```

This would give the same output. I made a simple demonstration of calling a function without parameters with just the selector here:
```sol
pragma solidity 0.8.30;

contract exampleContract {

    string coolString;

    function changeStringWithSelector() public {
        address(this).call(hex'dce5c757');
    }

    function cool() public {
        coolString = "I am cool";
    }

    function getString() public view returns (string memory) {
        return coolString;
    }
}


```
In this example, the function `changeStringWithSelector()` calls the function `cool()`, which takes no parameters. Because of this, it can be called without `abi.encodeWithSelector`.
