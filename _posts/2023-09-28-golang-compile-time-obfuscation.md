---
title: "Golang: Compile Time Obfuscation"
layout: post
excerpt_separator: <!--more-->
category: Blog
---

When starting out in defence evasion, one of the first things you learn is the value of even the simplest string obfuscation. Back when a lot of malware was written in C one of the easiest ways to implement this was this pre-processor macros which would run basic XOR operations on string literals. You can see a basic example of that [here](https://yurisk.info/2017/06/25/binary-obfuscation-string-obfuscating-in-C/), albeit they're using a Caesar cipher.

Unfortunately, for those of us that like to code in Go, it does not have support for pre-processor macros. So what are our options? Obfuscate the strings manually before compiling our tools? That's a pretty cumbersome workaround. Luckily for us, Go supports metaprogramming, which we can use to facilitate compile-time string obfuscation to get rid of those pesky static signatures.

<!--more-->

> **TL;DR** take a look at [github.com/redskal/obfuscatxor](https://github.com/redskal/obfuscatxor)

#### The Why
Anti-virus vendors have been known - from time to time - to base their static signatures on the existence of case-sensitive string literals found in malware. Some combinations of strings may also raise suspicions, such as the existence of several API function names commonly associated with a particular offensive technique. Even the most basic obfuscation may be enough to break these detections.

The ability to implement these basic protections should be early on the checklist of any self-respecting malware developer. So the question is, "how do we implement this in Golang?"

#### The How
Metaprogramming, in this instance, boils down to writing code that writes code for us. The Go compiler supports a `generate` command which is used to process `//go:generate` comments inside Go source code files. The format of these comments is as follows:

```go
//go:generate <some_command> <arguments> ...
```

When `go generate` is executed the commands will be run, which we can use to generate Go code. This is how we will generate our obfuscated code before we build our malware binary.

There are already tools which work in this way within the Golang ecosystem. The basis for my tooling was [mkwinsyscall](https://cs.opensource.google/go/x/sys/+/5a0f0661:windows/mkwinsyscall/mkwinsyscall.go) (or - more precisely - C-Sto's [modified version](https://github.com/C-Sto/BananaPhone/tree/master/cmd/mkdirectwinsyscall)), which is used to generate Win32api wrapper functions based on a prototype. With some modification, it was easy to implement string XOR and hashing using the same parsing techniques. Because a lot of the tooling is derivative of these previous tools I won't go over every line of code; I'll touch on the parts important to this project.

The codebase is going to be arranged like so:

<img src="/img/code-tree.png" class="article-image">

In order to generate Golang code, we first need to define what our files to be processed will look like. An example of the format for the current tool iteration:

```go
//go:generate go run github.com/redskal/obfuscatxor/cmd/obfuscator -output strings-out.go strings.go
package testing

//obfuscate Key(ObfuscateThis) Phrase(Just a test string) VarName(test1)
//obfuscate Key(ObfuscateMe) Phrase(Something, something, whatever, whatever) VarName(test2)

//hash phrase(hash me please) VarName(hash1)
//hash phrase(another thing to hash) varname(hash2)
```

The two main objectives of this tool will be to obfuscate strings and to hash strings. We'll be needing structures to store the results of each string in, so let's start with string obfuscation:

```go
type ObsString struct {
	Plaintext	string
	Key		string
	Varname		string
	Encrypted	[]byte
	EncryptedPretty	string
}
```

As we process each string, we can store the plaintext value, the XOR key, the variable name, and the processed values. This struct will allow us to use Golang's templating libraries to loop through and generate code efficiently. We'll need a similar struct for string hashing:

```go
type CRCHash struct {
	Plaintext	string
	Hash		string
	Varname		string
}
```

For the main functionality of this tool we want to do things very close to how `mkwinsyscall` has done it:
1. Iterate through source files
2. Generate a Go source file from templates
3. Write new file to disk

For the most part we can recycle previous code. We just need to modify the parsing and the output template. So let's start with the parsing. The first difference I have in the parsing process is in the [ParseFile](https://github.com/C-Sto/BananaPhone/blob/master/cmd/mkdirectwinsyscall/source.go#L82) function. As well, because our output is different to `mkwinsyscall`, we need a different output struct. This is what I came up with:

```go
type Source struct {
	String		[]*ObsString
	Hash		[]*CRCHash
	PackageName	string
}
```

The intention of this struct is to keep track of all the processed strings and the package name in order to generate the output file from our template.

Within my pre-processed files I'm using different markers to `mkwinsyscall`, so I've added some conditionals to cope with that and to indicate our intensions for the string:

```go
// not the droid you are looking for
if !strings.HasPrefix(t, prefixObfuscate) && !strings.HasPrefix(t, prefixHash) {
	continue
}

// we're not interested in the prefix
var obfuscate bool // used to judge between xor and crc hash
if strings.HasPrefix(t, prefixObfuscate) {
	t = t[len(prefixObfuscate):]
	obfuscate = true
} else if strings.HasPrefix(t, prefixHash) {
	t = t[len(prefixHash):]
	obfuscate = false
}
```

We then need to process the string in whichever way is intended. We process the string either by XOR or hashing, then append the result to our output struct.

```go
// process the string we have either through xor or hashing
if obfuscate {
	str, err := newString(t)
	if err != nil {
		return err
	}
	src.String = append(src.String, str)
} else {
	hsh, err := newHash(t)
	if err != nil {
		return err
	}
	src.Hash = append(src.Hash, hsh)
}
```

To give some context to the processing done above, let's take a look at the `newString()` function. We start off by creating an empty `ObsString`. We then use [extractSection](https://github.com/C-Sto/BananaPhone/blob/master/cmd/mkdirectwinsyscall/function.go#L29) to pull out the values we need and assign them to our `ObsString` values.

```go
// newString parses string s and returns an ObsString.
func newString(s string) (*ObsString, error) {
	s = strings.TrimSpace(s)

	str := &ObsString{}

	// extract the 3 strings
	var p string
	var b string
	var found bool
	for i := 0; i <= 2; i++ {
		p, b, s, found = extractSection(s, '(', ')')
		if !found || !containsWord(p) {
			return nil, errors.New("Could not extract information from \"" + s + "\".")
		}
		switch strings.ToLower(p) {
		case "key":
			str.Key = b
		case "phrase":
			str.Plaintext = b
		case "varname":
			str.Varname = b
		}
	}

	// we can't encrypt the string and produce output without these.
	if str.Key == "" || str.Plaintext == "" || str.Varname == "" {
		return nil, errors.New("key, variable name, or plaintext string not supplied")
	}

	// encrypt the string and get a pretty version
	str.Encrypted = []byte(obfuscate.StringXOR(str.Plaintext, str.Key))
	str.EncryptedPretty = prettifyBytes(str.Encrypted)

	return str, nil
}
```

We need a very similar function for `newHash()`, but we're going to generate a CRC64 hash of the string. The concept here was to use these for things like walking IATs, enumerating processes for AV/EDR, environmental keying (guard railing) checks, etc.

```go
// newHash parses string s and returns CRCHash
func newHash(s string) (*CRCHash, error) {
	s = strings.TrimSpace(s)

	crc := &CRCHash{}

	var p string
	var b string
	var found bool
	for i := 0; i <= 1; i++ {
		p, b, s, found = extractSection(s, '(', ')')
		if !found {
			return nil, errors.New("Could not extract information from \"" + s + "\"")
		}
		switch strings.ToLower(p) {
		case "phrase":
			crc.Plaintext = b
		case "varname":
			crc.Varname = b
		}
	}

	// Can't hash a non-existant string or delcare it if name is not known
	if crc.Plaintext == "" || crc.Varname == "" {
		return nil, errors.New("variable name or plaintext string not supplied")
	}

	// hash the string using ECMA CRC64 hash table
	uintHash := obfuscate.GetCRCHash(crc.Plaintext)
	crc.Hash = strconv.FormatUint(uintHash, 10)

	return crc, nil
}
```

Once everything has been iterated over and our `Source` instance is full of goodies, we need to generate the source code from a template. The `Generate()` function I'm using is very similar to C-Sto's, with some minor bits stripped out:

```go
// Generate output source file
func (src *Source) Generate(w io.Writer) error {
	// create our function map
	funcMap := template.FuncMap{
		"packagename": src.GetPackageName,
	}

	// process template and write to w
	t := template.Must(template.New("main").Funcs(funcMap).Parse(outTemplate))
	err := t.Execute(w, src)
	if err != nil {
		return errors.New("Failed to execute template: " + err.Error())
	}
	return nil
}
```

The template we're generating from will need to look like this:

```go
{% raw %}const outTemplate = `
{{define "main"}}// Code generated by 'go generate'; DO NOT EDIT

package {{packagename}}

var (
{{range .String}}// Key: "{{.GetKey}}", String: "{{.GetPlaintext}}"
{{.GetVarName}} = {{.GetPretty}}
{{end}}

{{range .Hash}}{{.GetVarName}} uint64 = {{.GetHash}} // String: "{{.GetPlaintext}}"
{{end}}
)
{{end}}{% endraw %}
`
```

Here, we're using Go's templating libraries to iterate through the `Source` instance to generate (hopefully) sensical output. The resulting Go file should look like this:

```go
// Code generated by 'go generate'; DO NOT EDIT

package testing

var (
	// Key: "ObfuscateThis", String: "Just a test string"
	test1 = []byte{
		0x05, 0x17, 0x15, 0x01, 0x53, 0x02, 0x41, 0x00, 0x00, 0x27, 0x1c, 0x49, 0x00, 0x3b, 0x10,
		0x0f, 0x1b, 0x14}
	// Key: "ObfuscateMe", String: "Something, something, whatever, whatever"
	test2 = []byte{
		0x1c, 0x0d, 0x0b, 0x10, 0x07, 0x0b, 0x08, 0x1a, 0x02, 0x61, 0x45, 0x3c, 0x0d, 0x0b, 0x10,
		0x07, 0x0b, 0x08, 0x1a, 0x02, 0x61, 0x45, 0x38, 0x0a, 0x07, 0x01, 0x16, 0x15, 0x04, 0x06,
		0x49, 0x6d, 0x12, 0x27, 0x03, 0x12, 0x10, 0x05, 0x06, 0x13}

	hash1 uint64 = 4933402316976831528	// String: "hash me please"
	hash2 uint64 = 15307073676716255198	// String: "another thing to hash"
)
```

To make use of this file we would include it into our new malware project, and import the `obfuscate` package. We can then use the package to "decrypt" the XOR'd strings, and hash values we've grabbed to compare to our stored values:

```go
import (
	"fmt"

	"github.com/redskal/obfuscatxor/pkg/obfuscate"
)

func main() {
	// "decrypt" our XOR'd string
	fmt.Println(obfuscate.StringXOR(string(test1), "ObfuscateThis"))

	// hash string and compare
	str := "hash me please"
	if hash1 == obfuscate.GetCRCHash(str) {
		fmt.Println("It's the same")
	} else {
		fmt.Println("Wrong-o!")
	}
}
```

The code within the `obfuscate` package is not complex. It serves only as a proof of concept that near-compile-time obfuscation can be done when using Golang for malware development. Take the code and implement something better. Just keep in mind the caveat of file entropy before going overboard with obfuscation.