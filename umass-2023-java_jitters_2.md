# UMass 2023 - java\_jitters\_2

**A LOT OF THE SOLUTION TO THIS PROBLEM IS REPEATED FROM THE FIRST `java_jitters` SO READ THAT WRITEUP BEFORE READING THIS ONE. YOU CAN FIND THAT WRITEUP** [**HERE**](umass-2023-java\_jitters.md)**.**

## Description

> _sips coffee_ if only i remembered the password to my amazing app... maybe i could get those java beans coins...
>
> **File**: [javajitters\_v2.jar](https://files.ivyfanchiang.ca/\_umassctf\_java/javajitters\_v2.jar)

## Assumptions from java\_jitters

Like the original java\_jitters challenge, we can unpack the JAR and decompile it with Recaf's version of Fernflower. The code is also obfuscated using the same Skidfuscator tool so the string literals encryption algorithm is the same (just with a new key that we need to trace). Like the original problem, there is a lot of `invokedynamic` instructions that you will need to convert to Java code by looking at the bytecode.

I also assumed that the `0o$Oo$K` and `K0o$KOo$KK` functions were the same SHA-256 password hashing functions from the original challenge and did not analyze those.

## Key differences from java\_jitters outside of main program logic

Like the original challenge, java\_jitters\_2 also uses exceptions to replace return statements but this time there are two exception classes. [`xpbyayedzpfnsdwh`](https://files.ivyfanchiang.ca/\_umassctf\_java/xpbyayedzpfnsdwh.java) replaces string return statements and [`edyxsdbugbromxsl`](https://files.ivyfanchiang.ca/\_umassctf\_java/edyxsdbugbromxsl.java) replaces byte array return statements.

This byte array return exception gets used in a new function called `Oo0o$OoOo$OoK(String var0, int var1, int var2)` which I renamed to `decode_data(String ciphertext, int len, int seed_arg)` for reasons I will discuss later.

Somewhat annotated/refactored version of the code available [here](https://files.ivyfanchiang.ca/\_umassctf\_java/Main2\_annotated.java)

## Analyzing the `main` function

Like the original problem, we can trace the program flow and the encryption key using JShell. The key is initialized in the same way as the last problem:

```java
int var3 = (new Random(187796278769191316L)).nextInt();
seed = 331542956 ^ var3;  // 694828334
int key = 232217141 ^ 2023401522 ^ seed;  // 1546112809
key = 1678878592 ^ key;  // 943089833
```

The program then proceeds to check the number of arguments supplied and then perform XOR and `Factory` checksum operations to change the key and then hashes the user's password.

The big difference between this program and the last is that instead of constantly checking hashes against known hashes in if-else blocks, we start by building a dictionary of known hashes and their respective message data (encoded in Base64):

```java
HashMap hash_to_base64 = new HashMap();  // dictionary of hashes and message data
key = 1550490048 ^ key;  // 1099337665
StringBuilder var177 = new StringBuilder("맳룺돵돳럺맷럹룺럺맴당맴맷맴룹돳돵돳럺맶럺맶맸돵돴럺룺돳돳럺럹맳룹럹맸룹돵맳돴럹맷맵당맵돸돵돸돵룹맴룺룹돵럹돴룺맷맵돳돴럺돵돵맳");  // encrypted version of known hash
int var178 = 0;
int var179 = 0;
​
// decrypting hash
for(key = 0 ^ key; var178 < var177.length(); var178 += 1099337664 ^ key) {
    var179 = var177.charAt(var178);
    var179 = (var179 ^ -1099337666 ^ key) & ((1099337664 ^ key) << (1099337681 ^ key)) - (1099337664 ^ key);
    var179 += 1099340170 ^ key;
    var179 ^= ((var179 >> (1099337668 ^ key) ^ var179 >> (1099337665 ^ key)) & ((1099337664 ^ key) << (1099337666 ^ key)) - (1099337664 ^ key)) << (1099337668 ^ key) | ((var179 >> (1099337668 ^ key) ^ var179 >> (1099337665 ^ key)) & ((1099337664 ^ key) << (1099337666 ^ key)) - (1099337664 ^ key)) << (1099337665 ^ key);
    var179 -= 1099335157 ^ key;
    var179 = ((var179 & (1099329598 ^ key)) << (1099337674 ^ key) | var179 >> (1099337668 ^ key)) & (1099329598 ^ key);
    var179 ^= 1099332967 ^ key;
    var177.setCharAt(var178, (char)var179);
}
​
String var36 = var177.toString();  // 30ec879082a2721cec84846eb80cc8931961e3b975a5fefe1201e9b075cb8ee3
StringBuilder var182 = new StringBuilder("怇帧帧慧捇捇执悧揧捇执愧捧枧捧慧悧帧座搇文座敧控悧崇扇掇揇敧揇掇揧敧座愧掇捧拧愧捧敧拇揧揇帧帧挧敇捇捧捇捧捇执挧揧揧惧敧掇揧敧悧敇捇惧捇揧敧愧挧捇崇拇揧捧捇揧文敇捧帧悧敧幧弧悧揧柇怇揇");  // encrypted version of message for hash
int var183 = 0;
int var184 = 0;
​
// decrypting the message to b64
for(key = 0 ^ key; var183 < var182.length(); var183 += 1099337664 ^ key) {
    var184 = var182.charAt(var183);
    var184 = ((var184 & (1099329598 ^ key)) >> (1099337676 ^ key) | var184 << (1099337666 ^ key)) & (1099329598 ^ key);
    var184 = ((var184 & (1099329598 ^ key)) >> (1099337673 ^ key) | var184 << (1099337673 ^ key)) & (1099329598 ^ key);
    var184 ^= 1099308569 ^ key;
    var184 += 1099349694 ^ key;
    var184 ^= 1099333357 ^ key;
    var184 -= 1099342553 ^ key;
    var182.setCharAt(var183, (char)var184);
}
​
String var2 = var182.toString();  // "cllfUUNXRUNdV0VfXlhCGhFPXkMWQFQWRFhdWVJdVFIRQllTEUVUVUNTRRZFWRFXEUZURFdTUkIRVURGEVlXFntXR1cQ"
hash_to_base64.put(var36, var2);  // storing the message data to the dictionary with hash as key
```

The program does this with 11 different sets of hashes and messages, changing the key every time. This gives us a dictionary that looks like this:

```json
{
    "8e09b341e9f7788d5563b587d6eb87dfa90069d668982dcbd19660bd08594564": "aFtEFFxBQkARXFBCVBRZVVUUUBRFRlhEXVEcR1lbRRRUR0FGVEdCWxFAXhRSRlBXWhRFXFhHEURQR0JDXkZVFQ==",
    "beae9a6258a7559ca2f8628763bcef3b44b126a093291af0532f217743bf2d52": "e1VHVRF+WEBFUUNHEVxQRxFZVEARXUVHEVlQQFJcEUNYQFkUSFtERhFEUEdCQ15GVRRSRlBXWl1fUxFHWl1dWEIV",
    "c7755b10c70d123e2082e7b21ef8efe3f400c5253fce315554726c5084c755e7": "ZVxUFHtVR1URQ15GXVARVl5DQhRVW0ZaEUBeFEhbRBRQWlUUSFtERhFEUEdCQ15GVRRSRlBXWl1fUxFHWl1dWEIV",
    "a76fb4e34b67a1fd1055c5bc78674ff627ec96763f4894b35c9a947ae0a6bc61": "e1VHVRF+WEBFUUNHEVVYWhZAEVNeQBFaXkBZXV8TEVtfFEhbRBRQWlUUSFtERhFXQ1VSX1haVhRCX1hYXUcQ",
    "a2c5009dbdd1a2a9935d34e8812257f1a965aa259f0efd9f5d2ecc5e3b809b03": "fVteX0IUXV1aURFNXkEWQlQUVltFFEVcVBR7VUdVEX5YQEVRQ0cRQV9QVEYRV15aRUZeWBFDWEBZFEVcWEcRRFBHQkNeRlUV",
    "30ec879082a2721cec84846eb80cc8931961e3b975a5fefe1201e9b075cb8ee3": "cllfUUNXRUNdV0VfXlhCGhFPXkMWQFQWRFhdWVJdVFIRQllTEUVUVUNTRRZFWRFXEUZURFdTUkIRVURGEVlXFntXR1cQ",
    "9affb91bb2f4f36e847b6bdcbd990d66035d7c0bede187a59ac56d98a21d4899": "aFtERhF+UEJQFFpaXkNdUVVTVBRYRxFWQ1FGXV9TEUBeFEFRQ1JUV0VdXloRQ1hAWRRFXFhHEURQR0JDXkZVFQ==",
    "eecd928fbae7909ec54cae3efc510470cb190f7c74ff0e3d87d00b26c5e76777": "aFtEE0dREVlQUFQUe1VHVRF+WEBFUUNHEVheW1oUXV1aURFQVFdQUhFDWEBZFEhbREYRRFBHQkNeRlUUUkZQV1pdX1MRRENbRlFCRxA=",
    "b0f8b56898e123c658a566bdd6d55e26a0b6cb0b5039fd53ef2f772c5ce2e3d5": "G0dYREIUUltXUlRRGw==",
    "8900fbb69012f45062aa6802718ad464eaea0854b66fe8916b3b38e775c296a8": "ZHlwZ2JPQwdHB0NHWFpWa1sARwBuBUJrBWtbBUVAAkZIa1sEU0k=",
    "6ceb89b10244f1d54471b0b6d595e78802252b7f269304ddb367440b966d988d": "aFtEE0dREUFfWF5XWlFVFEVcVBR7VUdVEUBDUVBHREZUFEZdRVwRQFldQhRBVUJHRltDUBA="
}
```

The program then checks if the hash from the user's password is found in the dictionary (which we can ignore when tracing in JShell) and then moves on to a message decoding and printing code block:

```java
PrintStream var33 = System.out;
String var54 = new String();
key = 1211708954 ^ key; // 1269109784
Decoder var4 = Base64.getDecoder();
String var75 = var10.toString();  // hash of input
Object var71 = hash_to_base64.get(var75);  // gets data matching hash
String var72 = (String)var71;  // b64 data
Charset var76 = StandardCharsets.UTF_8;
byte[] var73 = var72.getBytes(var76);
byte[] var69 = var4.decode(var73);
Charset charsetUTF = StandardCharsets.UTF_8;
String var3 = new String(var69, charsetUTF);  // b64 decoded data
byte var70 = (byte)(1269109784 ^ key);  // 0
String var66 = args[var70];  // password (unhashed)
key = 569683477 ^ key;  // 1783740941
int var67 = var66.length();  // password length
key = 1651564803 ^ key;  // 136403726
​
try {
    while(true) {
        decode_data(var3, var67, 133764025);
    }
} catch (edyxsdbugbromxsl var224) {
    byte[] var65 = var224.get();
    Charset var68 = StandardCharsets.UTF_8;
    var54.<init>(var65, var68);
    key = 858574774 ^ key;
    ymvjazxcysollnvc(var33, var54, 231835709);
    key = 1346937356 ^ key;
    return;
}
```

It starts by grabbing the message data for the hash provided, decoding the data with Base64, and then runs the `decode_data` function on the decoded data with the user's password length and `133764025` as arguments. From this we can deduce that `decode_data` is a decryption function for the message that takes password length as a key.

This is a pretty simple key to brute force which we can do for every message in JShell:

```java
jshell> str = "ZHlwZ2JPQwdHB0NHWFpWa1sARwBuBUJrBWtbBUVAAkZIa1sEU0k="
str ==> "ZHlwZ2JPQwdHB0NHWFpWa1sARwBuBUJrBWtbBUVAAkZIa1sEU0k="
    
jshell> for(int i = 1; i <= 256; i++) {
    ..>     try {
    ..>         decode_data(new String(Base64.getDecoder().decode(str.getBytes(StandardCharsets.UTF_8), StandardCharsets.UTF_8), i, 133764025);
    ..>     } catch (edyxsdbugbromxsl e) {
    ..>         System.out.println(new String(e.get()));
    ..>     }
    ..> }
```

This gives us all the messages in the program like `You must have had a triple-shot espresso to crack this password!` and `*sips coffee*`, but most importantly it also gives us the message that contains our flag: `UMASS{r3v3rsing_j4v4_1s_4_j1tt3ry_j0b}`

All the source code that was relevant for both java\_jitters and java\_jitters\_v2 in decompiled and annotated/refactored forms can be found [here](https://files.ivyfanchiang.ca/\_umassctf\_java/)
