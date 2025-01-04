---
layout: post
title:  "Family Generation - Pairings"
date:   2025-01-03 20:00:00 +1030
---

Continuing on from [generating names]({% post_url 2025-01-02-family-generation %})
the stage is pairing such that the children between the first families are
paired of together to create the next generation of families.

Components
==========
The goal is to generate several pieces of information.

* [Name generation]({% post_url 2025-01-02-family-generation %})
* Pairing (more widely known as marriage) - here.
* Date generation - next
* Place generation

Pairing
=======
The next stage is pairing so the children between the first families are paired
of together to create the next generation of families.

The children of the first family becomes the parents of the next generation.

## Inputs

The families in the current generation.

## Outputs

The family in the next generation where the parents of this generation are
children from the past generation.

## Constants

* Chance of being single - this should be about 0.2 (or 20%) in practice but
  that number turned out to be too high for the initial seeding. To account for
  that you would need many more people in the first generation or simply lower
  the number to 0.1 (10%). In the future, it might make sense that it starts
  off higher and then drops to 0.1 after many generations.

## Algorithm
* Filter out families without children
* For each child, determine if they are to remain single also known as being
  unpaired.
* For each child, if they not to stay single, add them to the list of
  potential male partners or female.
* For each male, try to find them a female
  * This excludes if the female is in the same family, i.e they are siblings.
    * This could be extend to prevent first or second cousin situations as well
      uncle/aunts if cross generation pairing occurs.
  * In the future a partner will be excluded if their is more than a 30 year
    age gap.
  * The selected female is removed from the list of potential candidates.
  * The new family is defined.
* The next step done is to generate children for the family.\
  This is same as it was done for the first families.
  * Generate how many children should be assigned to each of the new families
  * Generate the children for each new family.
  * If the parent's have date of birth, generate dates for the children.\
    This will be revisited later once the parent's are assigned birth dates.

## Comments
For the initial generation of pairings to lead to families, it is kept simple
by sticking if the partners are opposite sex so male and female instead of same
sex couple.

As you may realize, pairings don't happen between generation, something to
consider is there might not be an odd number of people or simply too many
people all from the one family left over after the pairings despite they were
selected for pairing (i.e. wasn't selected to stay single).

A future improvement would be to collect up the list of remaining
potential parents unpaired individuals and make them potential candidates
for the next generation.

At this point, the algorithm can be run again to compute the families in the
next generation and the process repeated to create the next generations.

## Example
This image shows two generations with only one child. The d3-dtree library
helped create this image.

![Graph showing three generations](/assets/2025-02-03-simple_graph.png)

Ideas
=====
- Compute the birth rate.
- Compute the number of paired people.
- Capture the unpaired individuals selected and make them candidates for
  pairing with the next generation. They still might not be suitable if
  their ages differ greatly.

Children
========
The first post overlooked how to select the number of children per family.

The starting point of this was the
[Count of dependent children in family (CDCF)](0) from the
[Australian Bureau of Statistics](1). Essentially, it provides information about
how many families have 0, 1, 2, 3, 4, 5 and 6 or more children within
Australia during a census (survey of the population).

The starting point for the weighting of how many children a family should have
was the census. The weights needed tweaking because the data is for 2020s and
does not produce good results when thinking about the 1950. The 6 or more group
gets broken down into 6, 7, 8  with the values split apart with the specifics
made up. The limit of 8 children per family.

Date generation
===============
The next stage is generating dates starting with the date of birth.

[0]: https://www.abs.gov.au/census/guide-census-data/census-dictionary/2021/variables-topic/household-and-families/count-dependent-children-family-cdcf
[1]: https://www.abs.gov.au/
