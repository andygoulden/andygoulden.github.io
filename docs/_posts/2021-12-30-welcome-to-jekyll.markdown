---
layout: post
title:  "An Open Letter to Spice8Rack"
date:   2021-12-30 19:48:10 +0000
categories: first
---

This is a response to a Spice8Rack video. In particular, this one:

[Magic: the Gathering has a Power and Toughness Problem](https://www.youtube.com/watch?v=TQi5zXTrL_I)

Particularly in particular, the section that starts around `10:50` into the video. Spice says this:

>Now, there are four-hundred and eight unique human soldiers in Magic: the Gathering including tokens at time of recording; and, for a long while, I tried to work out a way to calculate their average power and toughness that didn't involve me just creating a google sheet and writing them all out manually.
>
>Uh, but then, uh, I got frustrated, and then the lockdown happened, and then I drank a lot of rhubarb and ginger gin liqueur, and when I woke up, I had a google sheet where I had manually entered all of Magic's human soldiers' powers and toughnesses.

This seemed like a lot of effort, and it made me wonder if there was an easier way.

Spoiler alert: there is, otherwise I wouldn't have written this article. So, that's this article:

# __How to do data science on Magic: the Gathering cards__

This article comes to you in 5 parts:

1. The Problem
2. The Solution, at a glance
3. What is JSON?
4. What is Python?
5. The Solution, in depth

## The Problem

In his video, Spice wanted to know the average power of Human Soldier creatures, and the average power of Boars, to make a point about the powers and toughnesses of creatures in Magic.

To do this, he needed to calculate the total power of Human Soldier creatures, and the total toughness, and divide each by the number of creatures, to get the mean of each. He also did this for Boars, but I'm going to skip that for this article, since the process would be the same.

He went to the website Scryfall - a Magic: the Gathering database site - and did a search for cards with the types "human" and "soldier".

He now wanted to take all of the powers and toughnesses of the returned cards, add them all together, and divide them each by the number of results, to produce the average power and toughness of all human soldiers in Magic.

```
total power / number of human soldiers = average power
total toughness / number of human soldiers = average toughness
```

So what did he do?

A normal person would have given up.

Someone who writes code for a living would write some kind of script to do it.

Spice decided to enter each power and toughness for more than four hundred cards into a GoogleDocs spreadsheet. And then, because he was a little drunk, he buggered it up and had to do it again.

<img src="{{site.url}}/images/spice-memnarch.jpg" style="display: block; margin: auto;" />

The first thing to notice about this, is that it is completely fucking mental.

Let's take a look at the Scryfall page for one of the cards we want to work with:

<img src="{{site.url}}/images/akroan-phalanx-annotated.png" style="display: block; margin: auto;" />

Clearly, Scryfall has some kind of structure for the card data. The page doesn't just have an image, it has separate fields for type, and power and toughness.

To do the maths we want to do, we need to extract that data. But how? We could probably write some code to scrape the data we want from the website itself, but that would be time-consuming, and probably put a heavy load on Scryfall's servers. 

This would be especially wasteful if we downloaded web pages with lots of stuff that we don't want - like the image - just to ignore it and take info like power and toughness.

What we **really** want is to just get the raw data that they use to run the site. But they wouldn't just *give away* their precious data, would they...?

<img src="{{site.url}}/images/scryfall-bulk-data.png" style="display: block; margin: auto;" />

It turns out, they will! We don't care about multiple printings etc, so we'll grab 'Oracle Cards', which is a text file with one entry for each Oracle entry. Here's the page you can get it from:

[Scryfall API Documentation - Bulk Data Files](https://scryfall.com/docs/api/bulk-data)

## The Solution

Now we have the data, we just have to grab the information we need. Let's take a look at that data!

<img src="{{site.url}}/images/json-dump-raw.png" style="display: block; margin: auto;" />

Bloody hell... that looks like nonsense at first glance. At second glance, though, you can probably pick out bits that will be recognisable to a Magic player.

You can see things like `"name":"Static Orb"`, `"mana_cost":"{3}"`, and so on.

However, even for someone who knows what they're looking at, this is hideous, and basically unreadable.

This is **JSON**, or 'JavaScript Object Notation'. It's a cross-platform format for saving data to a file, known as 'serialisation'.

To get a better idea of what this file contains, we can use a JSON-parser to make the formatting a bit nicer. This is not required for the actual processing that we'll do later, but it will help as a visual aid. On most Linux distributions you can get a JSON-parser called `jq`, and use it to read in the file and print it out nicely like so:

```
jq -C '.' /path/to/your/file/oracle-cards-20211224100401.json  | less -r
```

That gives you something like this:

<img src="{{site.url}}/images/json-dump-pretty.png" style="display: block; margin: auto;" />

Now we can actually see what's going on!

The data is actually really well laid out, once you've indented it and added some colour.

Now, we just need to make some kind of computer instruction to do the search-y maths-y bit. To understand what we need to do with this file, we need to understand exactly what we have here.

## What is JSON?

As I said above, JSON is short for 'JavaScript Object Notation'. It started as part of the JavaScript language, but is now supported by many languages, like `C++`, `Ruby`, `Python`, and lots lots more.

It's just normal text (referred to as 'plain text'), meaning you could write or open a .json file using MS Notepad.

There's just six data types in JSON, and they can be grouped into 'values', and 'containers'.

A value is an actual data point. The value types are:

* **Number** - for numerical values, eg `5` or `1.1`
* **String** - for text, eg `"some words"`
* **Boolean** - for things that can only be `true` or `false`
* **Null** - a type to represent 'nothing', or 'not applicable'

A container is a group of things, which can be values and/or more containers. The container types are:

* **Array** - an ordered list of things, wrapped in square brackets, like `["Ajani", "Chandra", "Garruk", "Jace", "Liliana"]`
* **Object** - a set of key-value pairs, wrapped in curly brackets, like `{"name":"Storm Crow", "type_line":"Creature — Bird"}`

Let's work out how our JSON file is structured. Here's a snippet, with the middle of the lines cut out and most of the lines skipped for brevity:

```
[
  {"object":"card","id":" ... },
  {"object":"card","id":" ... },
  {"object":"card","id":" ... },
  {"object":"card","id":" ... },
  {"object":"card","id":" ... },
  {"object":"card","id":" ... },
  {"object":"card","id":" ... },
  {"object":"card","id":" ... },
...
  {"object":"card","id":" ... }
]
```

It starts and ends with a square bracket, which means the whole file is one array. That makes sense, because we can see that each line is a separate card.

Each line starts and ends with a curly bracket (plus a comma at the end for all but the last line), which means that each card is an object. That also makes sense, because a card will have lots of different bits of information about it.

Right, let's write some code...

## What is Python?

I chose **Python** as the language to write the code in.

Python is a programming language designed to be easy to read and easy to learn, but be powerful enough for most tasks that don't need to run especially fast.

There are many languages that this could be done in, but here are some of my reasons for picking Python:

* It's easy to pick up
* It runs on anything
* It's easy to set up the environment to run a script
* It's got a great built-in library for doing JSON stuff
* It's useful for lots of other things, so we're not picking up a niche skill that we'll never get a chance to use again

Actually learning Python would take too much time here (looking on YouTube, there's a 4-hour course, and a 1-hour 'beginner' course). Instead, I'll just show you the script in pieces, and tell you what it does.

If you want to follow along at home, I recommend getting `Visual Studio Code`, which is a text editor that runs on every platform, and has a great plug-in for Python which lets you run it in the same window as the code, without having to work out how to open and use a command window.

This guide is probably the best to get set up with VS Code and Python:

[Getting Started with Python in VS Code](https://code.visualstudio.com/docs/python/python-tutorial)

Right, let's look at the code...

## The Solution, in depth

Python scripts usually execute from the top and go line-by-line to the bottom. To make the code re-usable, I split it up into functions. The only function that runs at the top level is a function called `main`. Here's what it does:

* Get the arguments passed to the script
  * The location of the Scryfall JSON file, and
  * An optional flag to make it print out extra detailed output for debugging
* Read the JSON file, and convert it to a Python object
* Initialises the results
* For each card in the JSON, runs a function called `process_card_or_face`
  * If a card has multiple faces (like an Innistrad werewolf), the function is called on each face as if it were a separate card
  * This function checks that a card matches our criteria, and if so updates the results
* Prints out the results plus the analysis

Easy! Here's what that looks like:

```python
def main():
    # Get user arguments
    parser = argparse.ArgumentParser()
    parser.add_argument("json_file", type=str,
                        help="Path to the JSON file containing a dump of data from Scryfall")
    parser.add_argument('-v', "--verbose", action='count', default=0,
                        help="Verbosity level (call multiple times for higher verbosity, eg -vv")
    args = parser.parse_args()

    # Open JSON database file
    with open(args.json_file, 'r') as scryfall_json_file:
        oracle_all = json.load(scryfall_json_file)

    # Prepare results
    results = {
        "number_of_soldiers": 0,
        "total_power": 0,
        "total_toughness": 0
    }

    # Process each card (if a card has multiple faces, handle them each separately)
    for card in oracle_all:
        if "card_faces" in card.keys():
            # double-face card
            for face in card['card_faces']:
                process_card_or_face(face, results, verbose=args.verbose)
        else:
            # single-face card
            process_card_or_face(card, results, verbose=args.verbose)

    # Print results
    print(f"Found {results['number_of_soldiers']} soldiers.")
    print(f"Total power of soldiers:     {results['total_power']}")
    print(f"Total toughness of soldiers: {results['total_toughness']}")
    print(f"Average power of soldiers: {results['total_power'] / results['number_of_soldiers']}")
    print(f"Average toughness of soldiers: {results['total_toughness'] / results['number_of_soldiers']}")
```

If you wanted to re-purpose this script to do something else with the Scryfall data, this main could stay mostly the same, and you'd just change the results and the process function to do what you want.

Here's what the `process_card_or_face` function does:

* Check that the card (or face) has a type line - if not, skips it
* Check that the card is a creature - if not, skips it
* Check that the card is both a soldier and a human - if not, skips it
* Optionally prints out some info about the card (if the `-v` argument was given when running the script)
* Check that both the power and toughness are actual numbers (not, for example, "*")
* If the card passed all the checks, it's a valid result
  * We increase the number of soldiers counted by 1, 
  * add the power to the total power, and 
  * add the toughness to the total toughness

Pretty simple as well! Here's that function:

```python
def process_card_or_face(card_face:dict, output:dict, verbose=0) -> None:
    """
    Determine whether card (or card face) is a soldier with valid
    power/toughness, and if so update output variables
    """

    # Skip if card/face doesn't have a type line
    if "type_line" not in card_face.keys():
        if verbose:
            print(f"Card or face doesn't have type line")
        return

    # Ignore non-creatures
    if "Creature" not in card_face['type_line']:
        return

    # Ignore creatures which are not human soldiers
    if "Soldier" not in card_face['type_line'] or "Human" not in card_face['type_line']:
        return

    # Optionally print out card info
    if verbose:
        print(f"Soldier found:")
        print_card_info(card_face)

    # Make sure power and toughness are actual numbers (eg not '*')
    try:
        int(card_face['power'])
        int(card_face['toughness'])
    except ValueError:
        print(f"Non-integer power or toughness: {card_face['power']}/{card_face['toughness']}")
        return

    # If we got here, we have a valid soldier - update output variables
    output['number_of_soldiers'] += 1
    output['total_power'] += int(card_face['power'])
    output['total_toughness'] += int(card_face['toughness'])
```

You might notice that there's a function here that I haven't mentioned: `print_card_info`. That's just a debug function I wrote to print out some of the card's info in a nice way. It's in the script, which I've added as an appendix at the end.

Running the script on the Scryfall database (as I downloaded it) produces this output:

```
Non-integer power or toughness: */*
Non-integer power or toughness: */*
Non-integer power or toughness: */*
Non-integer power or toughness: */4
Found 443 soldiers.
Total power of soldiers:     902
Total toughness of soldiers: 1045
Average power of soldiers: 2.036117381489842
Average toughness of soldiers: 2.35891647855530
```

That means it skipped four creatures with stats it couldn't understand, leaving 443 human soldiers, with average power and toughness that round to `2/2`. Woo!

## Conclusion

This concludes my letter to Spice8Rack. 

Do I expect Spice to read this page, spend hours learning Python and JSON, and master the Scryfall API to the point where he could write a script like this himself?

No.

However, I hope that he can take the script I wrote, get it to run, and maybe hack it to do something useful for a future video. Or, failing that, he can send this article to someone else and get them to adapt it for whatever future Magic-madness he works on in the future.

Stay spicy,

Andy

## Appendix

As promised, the full script:

```python
#! /usr/bin/env python3

import argparse
import json
import sys


def print_field(card_object:dict, field:str) -> None:
    """Attempt to print a field, which may not exist"""
    try:
        print(f"{field}: {card_object[field]}")
    except KeyError:
        print(f"(no field '{field}')")


def print_card_info(card_object:dict) -> None:
    print_field(card_object, 'name')
    print_field(card_object, 'mana_cost')
    print_field(card_object, 'type_line')
    print_field(card_object, 'power')
    print_field(card_object, 'toughness')
    print("") # blank line


def process_card_or_face(card_face:dict, output:dict, verbose=0) -> None:
    """
    Determine whether card (or card face) is a soldier with valid
    power/toughness, and if so update output variables
    """

    # Skip if card/face doesn't have a type line
    if "type_line" not in card_face.keys():
        if verbose:
            print(f"Card or face doesn't have type line")
        return

    # Ignore non-creatures
    if "Creature" not in card_face['type_line']:
        return

    # Ignore creatures which are not human soldiers
    if "Soldier" not in card_face['type_line'] or "Human" not in card_face['type_line']:
        return

    # Optionally print out card info
    if verbose:
        print(f"Soldier found:")
        print_card_info(card_face)

    # Make sure power and toughness are actual numbers (eg not '*')
    try:
        int(card_face['power'])
        int(card_face['toughness'])
    except ValueError:
        print(f"Non-integer power or toughness: {card_face['power']}/{card_face['toughness']}")
        return

    # If we got here, we have a valid soldier - update output variables
    output['number_of_soldiers'] += 1
    output['total_power'] += int(card_face['power'])
    output['total_toughness'] += int(card_face['toughness'])


def main():
    # Get user arguments
    parser = argparse.ArgumentParser()
    parser.add_argument("json_file", type=str,
                        help="Path to the JSON file containing a dump of data from Scryfall")
    parser.add_argument('-v', "--verbose", action='count', default=0,
                        help="Verbosity level (call multiple times for higher verbosity, eg -vv")
    args = parser.parse_args()

    # Open JSON database file
    with open(args.json_file, 'r') as scryfall_json_file:
        oracle_all = json.load(scryfall_json_file)

    # Prepare results
    results = {
        "number_of_soldiers": 0,
        "total_power": 0,
        "total_toughness": 0
    }

    # Process each card (if a card has multiple faces, handle them each separately)
    for card in oracle_all:
        if "card_faces" in card.keys():
            # double-face card
            for face in card['card_faces']:
                process_card_or_face(face, results, verbose=args.verbose)
        else:
            # single-face card
            process_card_or_face(card, results, verbose=args.verbose)

    # Print results
    print(f"Found {results['number_of_soldiers']} soldiers.")
    print(f"Total power of soldiers:     {results['total_power']}")
    print(f"Total toughness of soldiers: {results['total_toughness']}")
    print(f"Average power of soldiers: {results['total_power'] / results['number_of_soldiers']}")
    print(f"Average toughness of soldiers: {results['total_toughness'] / results['number_of_soldiers']}")


if __name__ == "__main__":
    main()
```
