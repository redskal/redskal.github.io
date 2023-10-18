---
title: "Low Entropy Malware: Encoding Shellcode as Text"
layout: post
excerpt_separator: <!--more-->
category: Blog
---

Earlier in the year, I became somewhat obsessed with decreasing file entropy while obfuscating shellcodes. One of the best resources I found while researching was [this blog post](https://sekuro.io/blog/obfuscating-shellcode-entropy/) by Riley Kidd which got me looking at encoding shellcode as words. So, I automated it and added a few bells and whistles. This blog post will be about the concepts of the technique - not about providing a full, working PoC as I'm working on integrating this functionality into a new release of [Metsubushi](https://github.com/ByteJunkies-co-uk/Metsubushi).

<!--more-->

The additions I've made to this technique increase the amount of low entropy data included in the resulting loader/dropper. This is done by including random prose or poetry, and using that to perforate the obfuscated shellcode into chunks.

#### The Concept

Entropy is a measure of randomness, which can be used to indicate whether or not a file contains encrypted or obfuscated data. Malware defences often use entropy as one of many indicators that a file may be worth investigating further.

Shellcode - particularly when it's encrypted - typically results in a high entropy. Riley Kidd's blog post mentioned above does a good job of explaining a method to obfuscate shellcode as English words, resulting in a low entropy blob. I've built on Riley's technique to increase the obfuscation.

As an example, my wordlist of English words (which I'll touch on later) has a shannon entropy of 4.308199351168905 according to [CyberChef](https://gchq.github.io/CyberChef/#recipe=Entropy('Shannon%20scale')). A typical Metasploit 1kb shellcode .bin file has a score of 5.920806436431105, while Sliver's 15kb shellcode comes in at 6.2926193583750205. The closer we come to the top score of 8 the worse things are for us.

Ideally, this obfuscation technique would be applied as one of many layers, combining it with encryption or similar. The goal is to hide signatures away from any prying eyes...well, software.

#### The How

My technique relies on the use of several wordlists to create a hash table at random. These can be whatever you'd like; I used lists of English, Russian and Klingon words, Pokémon names, Disney characters, etc. The important part is that you have more than 256 entries per list (because a byte's value can be 0x00-0xff), and we want the hash table to be as random as possible each time it's created.

From the selected wordlist, 256 words should be selected at random to create the hash table. The flow would look something like this:

1. Open selected wordlist, reading each line into an array.
2. Create two maps/dictionaries: one keyed with byte value, one with the words.
3. Loop through taking 256 unique words at random, assigning the values to the maps from stage 2.

From this point, you can encode your shellcode by looping through each byte, using it's value to grab the corresponding word from the map keyed by byte values.

The shellcode needs to be broken up into smaller chunks which will then be interlaced with blocks of random prose or poetry. To do this I split it into 2kb chunks, and calculated the remainder for adjustments:

```go
codeChunkAmount := (shellcodeLength / (1024 * 2))
codeChunkSize := shellcodeLength / codeChunkAmount
codeChunkAdjust := shellcodeLength % codeChunkAmount
```

Now these values can be used to calculate how many blobs of random text we need to provision. Where do you get these pieces of text? These are some of the resources I used, but I'm sure there are more out there:

1. [APIQuotes](https://apiquotes.com)
2. [IT Bookstore](https://itbook.store)
3. [OpenLibrary](https://openlibrary.org)
4. [Poemist](https://www.poemist.com)

Each of these resources provide an API which can be used to collect random text for use in your decoder skeletons. If you structure this part modularly, other resources could be added easily. Using APIs does mean we need an internet connection to produce our skeletons, but that shouldn't be a huge drawback. Proper Planning and Preparation Prevents Piss Poor Performance, etc, etc.

Once you have your hash tables, encoded shellcode, and blobs of text the next step is to produce a skeleton for a loader or dropper. The layout I produced looked something like this:

```cpp
// our blend of text and encoded shellcode...
const string TEXT_PART1 = "Bunch of prose";
const vector<string> SHELLCODE_PART1 = { "word1", "word2", ... };
const string TEXT_PART2 = "More prose, or maybe some poetry";
const vector<string> SHELLCODE_PART2 = { "word5", "word3", ... };
const string TEXT_FINAL = "Some more text from somewhere...";

vector<unsigned char> decoder(vector<string> strings) {
    map<string, int> languageMap;
    languageMap["word1"] = 0;
    languageMap["word2"] = 1;
    // and so on...

    vector<unsigned char> matchedValues;
    for (int i = 0; i < strings.size(); i++) {
        if (languageMap.count(strings[i]) > 0) {
            int matchedValue = languageMap[strings[i]];
            matchedValues.push_back(matchedValue);
        }
    }
    return matchedValues;
}

int main() {
    vector<unsigned char> allParts;
    vector<unsigned char> currentPart;

    currentPart = decoder(SHELLCODE_PART1);
    allParts.insert(allParts.end(), currentPart.begin(), currentPart.end());
    // and so on...
}
```

For my PoC I used the Golang templating libraries to produce skeletons based on several different languages, which could be selected on the command line.

The produced skeleton can be adapted using your preferred flavour of loader or dropper tactics and techniques. My preference is to have an encrypted shellcode underneath this layer, which is environmentally keyed to provide some guardrails to detonation.

#### Polishing Your Encoder

If you looked at the above decoder snippet and thought "I can't code in C++," don't panic! I'm barely able to code C++, too. But ChatGPT can help to produce enough of a working template that can be debugged and refactored into working code.

One thing to note is that when producing Golang skeletons, keep in mind that the compiler will strip any unused variables from the compiled binary. So without some planning the compiler would strip the `TEXT_PARTn` variables from the produced loader.

#### Conclusion

This technique should be used as one of multiple layers of obfuscation. I think of it as a "defence in depth" approach - use this, encryption, anti-debugging, anti-sandboxing, etc. to achieve the goal. Also keep in mind this is aimed at static/run-time evasion - not in-memory evasion. So you're still going to have to put in work to customise implants enough to get over those hurdles.

When I finally get around to revisiting and refactoring Metsubushi, this technique will be one of the obfuscation techniques it offers. Until then I leave it as an exercise to the reader to fashion a working tool from this information.