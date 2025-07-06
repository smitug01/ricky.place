---
title: "Aegis CTF 2024 Writeup"
description: "This post contains the writeup for the Aegis CTF 2024 competition."
publishDate: "14 Sep 2024"
tags: ["aegis", "ctf", "writeup"]
draft: false
---

### WEB - 1. JSFBOX

題目中提供一個網頁，只能輸入符號 (Symbol)，有提供原始碼，
要讓網頁爆出 Flag。

#### 題目

```
const express = require('express');
const app = express();
const fs = require('fs');
const fa = fs.readFileSync('/flag', 'utf-8');
const port = 1024;

app.use(express.text());

app.use("/", express.static("static"));

app.post("/escape", (req, res) => {
    body = req.body;
    let validationResult = validateString(body);

    if (validationResult !== "String is valid.") {
        return res.send(validationResult);
    }

    console.log(body);

    let result;
    try {
        result = eval(body).toString();
    } catch (e) {
        return res.send("Something went wrong with your code.");
    }

    try {
        if (String (eval(result)) === fa) {
            return res.send("WOW! How did you know the flag?");
        }
    } catch (e) {}

    return res.send("Good job! Try harder.");
});

app.listen(port, () => {
    console.log(`App running on http://localhost:${port}`);
});

function validateString(input) {
    if (!/^[^\w=]+$/.test(input)) {
        return "ERROR: Input must only contain symbols.";
    }

    const charCount = {};
    for (let char of input) {
        charCount[char] = (charCount[char] || 0) + 1;
    }

    const sortedSymbols = Object.keys(charCount).sort((a, b) => charCount[b] - charCount[a]);

    if (sortedSymbols.length > 2**4) {
        return "ERROR: More than 8 different symbols used.";
    }

    for (let i = 0; i < sortedSymbols.length; i++) {
        const maxAllowed = 2**7 - i;
        if (charCount[sortedSymbols[i]] > maxAllowed) {
            return `ERROR: Symbol '${sortedSymbols[i]}' appeared ${charCount[sortedSymbols[i]]} times, which exceeds the limit of ${maxAllowed}.`;
        }
    }

    return "String is valid.";
}

```

#### 解法

使用 eval 函數對比，可以直接帶入 res.send(fa)，就能成功把 Flag 搞出來。

題目既然只能用 symbols 作為輸入，那第一步就聯想到 JSFuck，
但原生 JSFuck 產生器會有很多符號，需要自己改良跟運用他比較寬鬆的 regex。

```
(!![]+[])[+!+[]] + // r
(!![]+[])[!+[]+!+[]+!+[]] + // r
(![]+[])[+!+[]+!+[]+!+[]] + // s
"." + // .
(![]+[])[+!+[]+!+[]+!+[]] + // s
(!![]+[])[!+[]+!+[]+!+[]] + // e
([][[]]+[])[+!+[]] + // n
([][[]]+[])[!+[]+!+[]] + // d
"(" + // (
((![]+[])[+[]]+(![]+[])[+!+[]]) + // fa
")" // )
```

### MISC - 1. Eazy Jail

題目有分兩個 Stage, 分別是 Python 跟 JS，
第一個 Stage，是在 Python int(input) 的情況下，
要分別用 1, 3, 4, 6, 7, 8, 10, 11 長度的輸入解出 2。

第二個 Stage，是在 JS 的請況下，
要同時滿足 Number(input) 跟 safeEval(input) 分別為 1024 與 532。

#### 題目

```
import os

def validate_and_execute(user_input, expected_length):
    EXPECTED_RESULT = 2
    allowed_characters = set('abcdefg123456{}"')

    if len(user_input) != expected_length:
        print(f"Invalid input length. Your input must be exactly {expected_length} characters long.")
        return False

    if set(user_input).difference(allowed_characters):
        print("Invalid input. Only a,b,c,d,e,f,g,1,2,3,4,5,{,} certain characters are allowed.")
        return False

    expression = f"int({user_input})"
    result = eval(expression, {'__builtins__': None}, {'int': int})
    try:
        if  result != EXPECTED_RESULT:
            print(f"Error.")
            return False
        else:
            return True
    except Exception:
        print(f"Error.")
        return False

levels = [1, 3, 4, 6, 7, 8, 10, 11]

def start_game():
    logo="""
░█████╗░███████╗░██████╗░██╗░██████╗ Eazy Jail For Beginner
██╔══██╗██╔════╝██╔════╝░██║██╔════╝
███████║█████╗░░██║░░██╗░██║╚█████╗░
██╔══██║██╔══╝░░██║░░╚██╗██║░╚═══██╗
██║░░██║███████╗╚██████╔╝██║██████╔╝
╚═╝░░╚═╝╚══════╝░╚═════╝░╚═╝╚═════╝░ STAGE 1"""
    print(logo)
    print("Your task is to enter a string that, when used in a command like 'int(YOUR_INPUT)', results in the number 2.")
    print("Each level requires a different input length. If you succeed through all levels, you will reach the next stage!")
    for level in levels:
        print(f"\nLevel {levels.index(level) + 1} of {len(levels)}: Enter a payload exactly {level} characters long.")
        while True:
            user_input = input("Enter a payload: ")
            if validate_and_execute(user_input, level):
                print(f"Congratulations on passing level {levels.index(level) + 1}!")
                if level == 11:
                    os.system('clear')
                    os.system('node stage2.js')
                    return
                break
            else:
                print("An error occurred.")
                return

if __name__ == "__main__":
    start_game()
    exit()
```

```
const fs = require('fs');
const fa = fs.readFileSync('./flag', 'utf-8');

const readline = require('readline').createInterface({
    input: process.stdin,
    output: process.stdout
});

const QUESTION = `
What's my favorite number?

- 1024 is my birthday
- 532 is my last 3 digits of my phone
`;

const CORRECT_ANSWER = 1024;
const EXPECTED_RESULT = 532;

console.log(QUESTION);

readline.question('Guess the number: ', input => {
    const guessedNumber = Number(input);
    console.log(guessedNumber);

    let spinnerInterval;
    let spinnerIndex = 0;
    const spinnerChars = ['|', '/', '-', '\\'];

    const startSpinner = () => {
        spinnerInterval = setInterval(() => {
            process.stdout.write(`\rChecking your answer... ${spinnerChars[spinnerIndex]}`);
            spinnerIndex = (spinnerIndex + 1) % spinnerChars.length;
        }, 100);
    };

    const stopSpinner = () => {
        clearInterval(spinnerInterval);
        process.stdout.write('\rChecking your answer... Done!\n');
    };

    startSpinner();

    setTimeout(() => {
        stopSpinner();
        console.log(`\n=====================================\n`);
        if (guessedNumber === CORRECT_ANSWER) {
            console.log(`Great! You got the number - Awesome!`);
            try {
                const evaluationResult = safeEval(input);
                if (evaluationResult === EXPECTED_RESULT) {
                    displaySuccess();
                } else {
                    displayFailure();
                }
            } catch (error) {
                displayError();
            }
        } else {
            displayFailure();
        }
        readline.close();
    }, 1500);
});

function safeEval(input) {
    const allowedChars = /^[0-9+\-*/.\s]+$/;

    if (!allowedChars.test(input)) {
        throw new Error('Invalid characters detected!');
    }

    return new Function(`return (${input})`)();
}

function displaySuccess() {
    console.log(`Good job! You got it! :)`);
    console.log(`Flag: ${fa.trim()}\n`);
}

function displayFailure() {
    console.log('Oops! Wrong answer! Try again :)\n');
}

function displayError() {
    console.log(`Error occurred! Please try again!\n`);
}
```

#### 解法

第一個 Stage: [ 2, "2", b"2", f"{2}", """2""", f"""2""", f"""{2}""", f"{f"{2}"}" ]

第二個 Stage: 運用新版 JS Number(input) 不會辨識進位的特性，使用 01024 讓他取得 1024, 後面 safeEval(input) 時會辨識進為八進位，1024 即為 532，可滿足兩個條件。
