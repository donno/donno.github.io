---
layout: post
title:  "Family Generation - Names"
date:   2025-01-02 20:00:00 +1030
---

Back in 2020, I started writing down ideas for a game idea. The idea of this
post while quite the tangent from the original idea was something I wanted
to work on. That idea was to generate families to live in a town as well as
businesses for those families to visit. A lot of that came from watching a
video on Dwarf Fortress.

The actual coding of this took place around 2023-09-23. This si therefore not
so much a journal of the steps I took but more a description of the system that
was built. I will add several months earlier I had looked at using
OpenStreetMap to source a list of shops and offices as a starting place for
businesses.

I doubt the original idea and thus whole game will ever be completed.  There is
a chance that parts of it to are turned into a different but smaller game, so
only time will tell there if anything comes of this.

Components
==========
The goal is to generate several pieces of information.

* Name generation
* Pairing (more widely known as marriage)
* Date of birth
* Date of death
* Date of marriage (maybe easier to do during pairing).
* Place of birth
* Place of marriage

Name generation
===============
There are three sets of lists that the names come from:
- Male given names
- Female given names
- Surnames

These lists contain the names and number of occurrences. The sources of
this information is as follows.

## Given names
* This data comes from popular baby names in the state of South Australia
  from 1944 to 2013.
* The data is licenced under the [Creative Commons Attribution](3).
* The data source is the [Popular Baby Names](0) from Data.SA.

## Surnames
* This data comes from deceased estate files of the state of New South Wales
  from 1880 to 1923.
* The data is licenced under the [Creative Commons Attribution](3).
* The data source is the [Deceased Estate Files, 1880-1923](1) from Data.NSW.

## Alternate
The original approach taken was to look at the ticket of leave data, which
contains a list of first names and surnames.

Essentially, a ticket of leave allowed a convict to work for themselves but
couldn't leave the colony so kind of more like a town-arrest rather than
house-arrest arrangement.

* This data is based on convicts in the state of New South Wales from
  1810 to 1875.
* The data is licenced under the [Creative Commons Attribution](3).
* The data source is the [Convict Index](2) from Data.NSW.

## Preparation
The list of baby names is divided by male and female and per year. A tally of
all of the female and male names are built up and saved.

The surnames are only a single file from 1810 to 1875. The names themselves
needed some minor adjustments improve the naming style before building up a
tally.

* Adjusting the style of the name so its title case rather than all upper case.
* Adjusting MC <name> to Mc<name>
* Adjusting O <name> to O'<Name>
* Removing the " ?" at the end of names.

As typing this up, the letter after Mc should be capitalised so it is
McClintock not Mcclintock, that could do with some extra tweaking.

### Sample - Female
```
Susan,7234
Sarah,7176
Jessica,6552
Jennifer,6481
Christine,6396
```

### Sample - Male
```
David,19714
Michael,18742
Peter,16973
John,13462
Andrew,13169
```

### Sample - Surname
```
Brown,1638
Smith,1563
Jones,1509
Campbell,901
Anderson,750
```

## Algorithm
Use `random.choices()` from Python to select N names taking into account the
cumulative weights. The weights are simply the tally of the counts from the
file. If the source file was: [Brown, 120], [Smith, 100], [Jones, 80] then the
cumulative weights are 120, 220, and 300 respectively.

The weights means that instead of each name having an equal probability they
are weighted based on their counts or occurrences.

These cumulative weights could have been pre-computed during the preparation
stage rather than at runtime, while the Python implementation can compute the
cumulative weights from just the weights it redundant work if you choosing
names each time.

The choice of an element doesn't reduce the probability of the same element
is chosen again.

### Inputs for choices()

* Population - the list of elements to choose from. In this case they will be
  the list of surnames.
* Cumulative weights - the list of weights where each subsequent weight
  includes the weights before it.
* Thee number of choices to make

### Formula for choices()

* `total = cumulative_weights[-1]`
* `upper = len(population) - 1`
* For i = 0 to k
    * `chosen_weight = random() * total`
    * `chosen index = bisect(cumulative_weights, chosen_weight, 0, upper)`
    * Yield  `population[chosen_index]`

The `bisect(array, value)` returns the index for the insertion point for where
the value belongs in the array. This is done by performing a binary search to
hence the bisect part. With the weighting above being [120, 220, 300] and the
random number of 180 then 180 would logically go between 120 and 220 and give
index 1, which in turn means that Smith is chosen because the names were
[Brown, Smith, Jones].

## Generate first families
The initial set of families generated are therefore the first families. The
people that form the families won't have any parents and thus ancestors of
their own, or you could think of it as being the records were lost to time and
are not recorded.

For the first families this is simple as both people are given the same
surname. In the future children could be given one of their parent's names
or both for a double barrel surname.

Assuming we want to generate k families then:
* Choose k female names
* Choose k male names
* Choose k surnames.
* Choose how many children each of the k families will have.
    * For each child chosen decide if they are male or female.\
      The chance of a child being male was based on the number of males verses
      females born in 2021 within Australia from [Births, Australia][4] by
      the Australian Bureau of Statistics. The number used was
      `0.5126420986077239`
    * Generate n boy names and m girl names.
    More on this later as for now the focus is purely on name generation.

The titles become are Mr and Mrs for parents and Master and Miss for children.

### Example
The parents only, these names were generated on 2024-09-23 when progress on
this project really picked up.

* Mr Adrian and Mrs Katherine Ballard
* Mr Seth and Mrs Virginia Loder
* Mr Maxwell and Mrs Marcia Leech
* Mr Daniel and Mrs Siobhan Andrews
* Mr Wayne and Mrs Elizabeth Moxey
* Mr Brad and Mrs Lauren Brodie
* Mr Lance and Mrs Maria Turnbull
* Mr Dimitrios and Mrs Kylie Jones
* Mr Jack and Mrs Kelly O'Neill

With children
* Mr Benjamin and Mrs Jo-Ann Johnstone
   * Miss Amy Johnstone
   * Master Michael Johnstone
   * Master Ricky Johnstone
* Mr Paul and Mrs Elizabeth Colvin
   * Miss Ada Colvin
* Mr Peter and Mrs Faye Mcguire
   * Miss Danni Mcguire
   * Master Denis Mcguire
* Mr Aaron and Mrs Hayley Lyne
   * Master Troy Lyne
   * Miss Carly Lyne
* Mr Mitchell and Mrs Julie Burt
* Mr Colin and Mrs Frances Beatty

The next part was paring which about two hours after the above I had a proof of
concept where it would do the pairing and printout:
```
Mr Trevor and Mrs Cheryl (née Burden) Lawson
```
Where that meant Master Trevor Lawson ended up marrying Miss Cheryl Burden
Lawson.

A graph was produced of the initial families and first set of marriages between
the families in GraphViz but it was not pretty.

Pairing
=======
The next stage is pairing so the children between the first families are paired
of together to create the next generation of families.

Hopefully, I will get around to writing about the pairing and possibly the
date assignment in my next post. Otherwise, it could be an overview of the
timeline as more of a journal of work done and when.

[0]: https://data.sa.gov.au/data/dataset/popular-baby-names
[1]: https://data.nsw.gov.au/data/dataset/deceased-estate-files-1880-1923
[2]: https://data.nsw.gov.au/data/dataset/convict-indexes
[3]: http://creativecommons.org/licenses/by/4.0
[4]: https://www.abs.gov.au/statistics/people/population/births-australia/latest-release