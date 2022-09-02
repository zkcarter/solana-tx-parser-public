## Generated docs
[Generated module docs](/docs/README.md)


## Table of contents
  * [Installation](#installation)
  * [What this tool can be used for](#what-this-tool-can-be-used-for)
  * [How To Use](#how-to-use)
    + [Parse by IDL](#parse-by-idl)
    + [Parse with custom parser](#parse-with-custom-parser)
- [More details](#more-details)
  * [Instruction deserialization](#instruction-deserialization)
  * [CPI flattening](#cpi-flattening)
  * [Parsing transaction logs](#parsing-transaction-logs)
  * [Using everything together](#using-everything-together)

### Installation
At the moment we haven't published this package to the NPM. But you can easily use this package in your projects using npm github installation: 

`npm i git+https://github.com/debridge-finance/solana-tx-parser-public`.

Which will produce following line in package.json dependencies section: 

`"@debridge-finance/solana-transaction-parser": "github:debridge-finance/solana-tx-parser-public"`

### What this tool can be used for
- Parse solana instructions using anchor IDL/custom parsers.
- Parse `SystemProgram`, `TokenProgram`, `AssociatedTokenProgram` instructions out of the box.
- Parse transaction logs.
- Convert ParsedTransaction/CompiledTransaction/base64-encoded transaction into a list of TransactionInstructions and parse it.
- Unfold transaction with CPI into artificial transaction with CPI calls included as usual TransactionInstruction.

### How To Use
#### Parse by IDL
First step: init parser
```ts
import { Idl } from "@project-serum/anchor";
import { PublicKey, Connection } from "@solana/web3.js";
import { SolanaParser } from "@debridge-finance/solana-transaction-parser";
import { IDL as JupiterIdl, Jupiter } from "./idl/jupiter"; // idl and types file generated by Anchor

const rpcConnection = new Connection("https://jupiter.genesysgo.net");
const txParser = new SolanaParser([{ idl: JupiterIdl as unknown as Idl, programId: "JUP2jxvXaqu7NQY1GmNF4m1vodw12LVXYxbFL2uJvfo" }]);
```
Second step: parse transaction by tx hash:
```ts
const parsed = await txParser.parseTransaction(
	rpcConnection,
	"5zgvxQjV6BisU8SfahqasBZGfXy5HJ3YxYseMBG7VbR4iypDdtdymvE1jmEMG7G39bdVBaHhLYUHUejSTtuZEpEj",
	false,
);
```
Voila! We have a list of TransactionInstruction with instruction names, args and accounts.
```ts
// we can find instruction by name
const tokenSwapIx = parsed?.find((pix) => pix.name === "tokenSwap");
// or just use index
const setTokenLedgerIx = parsed[0] as ParsedIdlInstruction<Jupiter, "setTokenLedger">;
```
#### Parse with custom parser
What if the instructions we want to parse do not belong to Anchor program? 
We could provide custom instruction parser to SolanaParser!
Custom parser need to have following interface: `(instruction: TransactionInstruction): ParsedCustomInstruction`.
We've implemented a small program for testing purposes which can perform two actions: sum up two numbers (passed as u64 numbers) OR
print provided data to log (passed as ASCII codes and as a second account in accounts list).

| First byte | Action                                                                           |
| ---------- | -------------------------------------------------------------------------------- |
| 0          | Print remaining data as text + string "From: " + second account pubkey as base58 |
| 1          | Sum up two numbers and print the result to log                                   |

Parser will be really simple in such case:
```ts
function customParser(instruction: TransactionInstruction): ParsedCustomInstruction {
	let args: unknown;
	let keys: ParsedAccount[];
	let name: string;
	switch (instruction.data[0]) {
		case 0:
			args = { message: instruction.data.slice(1).toString("utf8") };
			keys = [instruction.keys[0], { name: "messageFrom", ...instruction.keys[1] }];
			name = "echo";
			break;
		case 1:
			args = { a: instruction.data.readBigInt64LE(1), b: instruction.data.readBigInt64LE(9) };
			keys = instruction.keys;
			name = "sum";
			break;
		default:
			throw new Error("unknown instruction!");
	}

	return {
		programId: instruction.programId,
		accounts: keys,
		args,
		name,
	};
}
```
Now we need to init SolanaParser object (which contains different parsers that can parse instructions from specified programs)
```ts
const parser = new SolanaParser([]);
parser.addParser(new PublicKey("5wZA8owNKtmfWGBc7rocEXBvTBxMtbpVpkivXNKXNuCV"), customParser);
```
To parse transaction which contains this instructions we only need to call `parseTransaction` method on parser object
```ts
const connection = new Connection(clusterApiUrl("devnet"));
const parsed = await parser.parseTransaction(connection, "2QU8jyEde9qbvtrYBJJZ2iBubqodmQRSoq2pfomHdGYgTgXwuncappiet8ojGGRdEkzkhW8sXdyfCxwuGHaHYegC");
// check if no errors was produced during parsing
if (!parsed) throw new Error("failed to get tx/parse!");
console.log(parsed[0].name); // will print "echo"
```

## More details

Main functions of this project are:
- instruction deserialization
- *flattening* transactions with CPI calls
- parsing transaction logs

### Instruction deserialization
This part is pretty simple: we have some `const mapping = Map<string, Parser>` where keys are program ids and values are parsers for corresponding programId - function that takes TransactionInstruction as input and returns ParsedInstruction (deserialized data, instruction name and accounts meta [with names]).
We can deserialize **TransactionInstruction** using Anchor's IDL or custom parser, hence we can deserialize everything which consists of (or can be converted into) **TransactionInstructions**: Transaction, ParsedTransaction, TransactionResponse, ParsedConfirmedTransaction, Message, CompiledMessage, CompiledInstruction or wire transaction, etc.
Steps of parsing are following:
1. Convert input into TransactionInstruction, lets name it **ix** (different functions need to be called for different input formats)
2. Find `ix.programId` in the mapping. If parser exists pass **ix** to it
3. Check parsed instruction name, set correct data types using generics

### CPI flattening 
Function: [flattenTransactionResponse](./src/helpers.ts#L87)  
Can be only done with TransactionResponse/ParsedTransactionWithMeta objects because we need `transaction.meta.innerInstructions` field.
`transaction.meta.innerInstructions` is a list of objects of following structure: 
```ts
export type CompiledInnerInstruction = {
  index: number, // index of instruction in Transaction object which produced CPI, 
  instructions: CompiledInstruction[]  // ordered list of instructions which were called after instruction with index *index*
}
```
We create artificial `const result: Transaction = new Transaction();` object which contains all the instructions from TransactionResponse + CPI calls:
Add first TransactionInstruction to result, check if CPI calls with index = 0 exist, add them to result, move to the next instruction, repeat.
Finally, we check that `result.instructions.length === input.instructions.length + total number of CPI instructions`.  
We can call index of result.instructions **callId** - index of call in the whole transaction. Same **callId** will be used in the logs part

### Parsing transaction logs 
Function: [parseLogs](./src/helpers.ts#L143)  
Working with Solana's logs is not a trivial task - to determine which program emitted current log line we have to restore call stack, check call depth and set correct **callId** for each log line. parseLogs function implements all that stuff (with call depth and call id checks): 
1. Iterate over logs
2. Check log type (invoke/return/error/program log/program data) using regex
3. According to the log type perform action:
   - Invoke: init and save new context object which contains program id, call depth of instruction, **callId**, index of instruction in Transaction which produced log (depth == 0) or current CPI call (depth != 0), save call stack (push current callId into stack)
   - Return success/fail: pop caller id from call stack, current callId = popped callId
   - program log/program data/program consumed: save log into context object with **callId** == current callId
4. Return list of context objects

### Using everything together
```ts
import { ourIdl } from "programIdl";

const parser = new SolanaParser([{programId: "someBase58Address", idl: ourIdl}]);
const flattened = flattenTransactionResponse(response);
const logs = parseLogs(response.meta.logs || []);
const parsed = flattened.map((ix) => parser.parse(ix));

const callId = parsed.findIndex( (parsedIx) => parsedIx.name === "someInstruction" );
if (callId === -1) return Promise.reject("instruction not found");

const someInstruction = parsed[callId];
const someInstructionLogs = logs.find((context) => context.id === callId);

const onlyNeededProgramLogs = logs.filter((context) => context.programId === "someBase58Address");
```
