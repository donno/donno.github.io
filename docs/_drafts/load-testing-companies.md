
Last year, 2025, I was tasked with testing how our web service at work
performed when under load. To confirm that the actions of users in one company
didn't cause problems for another company, I set out to create thousands of
users and tens of company for those users to work at.

Firstly, it should be mentioned that the recommended approach is typically to
use the same user account for the tests, with the idea being 1 user doing
100 actions per second is the same as 100 users doing 1 action per second.
I was concerned that the multi-tenanting aspect of the system wouldn't hold up
and thus wanted to make sure that if users from another company were accessing
the system the experience of the users of other companies weren't degraded.

Entities to generate:
- Users
    - First name
    - Last Name
    - Email - This is based on the company email policy and their name.
- Groups
- Companies
    * Name
    *  Domain Name
    * User Name Style
        * first_name.last_name
        * First letter of first name and full last name.
        * Up to first three characters for first name, a dot and then full last name.
        * Up to first four characters for first name and first four of last name.
        * Assign the employee a number and use that.

## User Names
To generate the names of the users, the code written for the family generation
project was rewritten.

* First names come from [popular baby names][1] in the state of South Australia
  from 1944 to 2013.
* Surnames of people who passed away between 1880 and 1923 from the
  [deceased estate files][2] of the state of New South Wales from 1880 to 1923.
* Key change was the first names weren't split into male and females.

[Code][name_people.py]

Example
```
Nikolaos Price
Mark Brewer
Naomi Burgess
Giuseppe Daly
Carrin Young
Maria Kelly
Deanne Whelan
Colby Hall
Kevin Currell
Georgia Doyle
```

The previous time my employee did load testing, they simply generated UUID and
split it into first and last names. This has the benefit of not looking real
but has the disadvantage of not being memorable. It a lot easier to relate to
when you see "Mark Brewer" in the user name field of a request then the UUID.
The other thing is that can often look like a bug that the actual UUID of a
user has leaked into the user's name field.

## Emails

Considering the names above various naming styles for emails would be:

* first_name.last_name
    * nikolaos.price
    * mark.brewer
    * naomi.burgess
* First letter of first name and full last name.
    * nprice
    * mbrewer
    * nburgess
* Up to first three characters for first name and full last name.
    * nik.price
    * mar.brewer
    * nao.burgess
* Up to first four characters for first name and first four of last name.
    * niko.pric
    * mark.brew
    * naom.burg
* Assign the employee a number and use that.
    * e1234
    * e1235
    * e1236

There were a few gotcha here as some names had multiple names and ended up
with spaces which were replaced with underscores and last names sometimes had
`'` so they were also removed.

## Company Names
The three sources of potential names that were  considered were:
* Genus of birds
* Names of prokaryotics
* Names of Australia suburbs

In the end, the I used Australian suburbs, which is the
[State Suburbs ASGS Edition 2016][3] in CSV format from the Australian
Bureau of Statistics.

Some initial clean-up of the names is done to remove some unwanted artifacts,
for example:
- "ACT Remainder - Cotter River"
    * Drop "ACT Remainder - "
- "Falls Creek (NSW)"
    * Drop the (State) part in the name, for example, " (ACT)"

I would have liked to have added option to customise the names to add a
prefix or a suffix but didn't end up doing that. THe idea being was to have
"Brothers" or "And Co" or "Limited".

[Code][name_company.py]

## User Generator
Generate company, groups and users, with the goal of creating fictional:

- Company / Organisations
- Groups / Departments / Roles
- Users / Employees

### Output Format
The output format is a JSON doucument where it is:
```json
{
  "companies": [
     <company>
  ]
}
```

Where company is: (See `CompanyJson`)
```json
{
   "name": str,
   "domain": str,
   "groups": [],
   "employees": [
       <employee>
   ]
}
```

Where employee is: (`EmployeeJson`)
```json
{
    "first": "John",
    "last": "Smith",
    "email": "john.smith@company.example",
    "groups": ["management"],
}
```

### Typed Output Format
The following is the same as above but using Python's `TypedDict` to define
the fields.
```python
import typing

class EmployeeJson(typing.TypedDict):
    """Represent an employee in JSON as a Python dictionary."""

    first_name: str
    last_name: str
    email: str
    group: list[str]
    """The name of the groups that the employee is a part of at their company.
    """


class CompanyJson(typing.TypedDict):
    """Represent the company in JSON as a Python dictionary."""

    name: str
    """The name of the company."""

    domain: str
    """The domain name for the company."""

    user_name_style: str
    """The name of the style for the user names as used in the emails."""

    employees: list[EmployeeJson]
    """The employees that work at the company."""

    groups: list[str]
    """The name of the groups (or roles) used at the company."""
```

### Group Allocation

The idea of a group was to either be based on role and/or department, so
for a large company, you may have "project A", "project B" and "project C" and
each project has a project manager, release engineer, testing lead, senior
developer, assuming you are modelling a software company.

Based on the company size, it decides what roles are available and what
weightings there are between the roles.

For example, for a company of 30 employees or less, the groups I used were
senior (2), manager (2) and clerk (16).

Where as for a company of 50, there is instead junior (8), specialist (10),
senior (4) and manager (3).

For companies with 200 or more, there is executive managers, team managers,
department heads (for one of 4 departments), senior specialists. specialist
where the specialists are split by department as well.

## Emails

When a company is generated, the user name style is chosen and the companies
domain is generated.

Domain generation is simply take the name, remove spaces, lower case it and
then use a `.example` domain. This domain makes it clear the domain and thus
email addresses are only examples and not valid addresses to send an email
to.

There were bigger ambitious here of of including a country code, but there
was no open data source that listed top-level domains (for countries) and
the number of registered domains for each to be able to use as a weight.

A post-processing step is performed after all the employees of a company are
generated to ensure each email is unique, by adding a random number between
10 and 200 before the `@`. This is mainly a problem for the first letter of
first name and full last name case where the clash rate is higher for larger
companies, likewise for the four from first and four from last name.

## All together

The end result is a list of companies, the groups at those companies and
the employees of the company. Really, it more the user accounts for the
employees of the company.

At this point, I was able to write a script which essentially registered
the companies, created the groups and registered a user account for each
employee at the companies in the system.

The [code][user_generator.py] for this is on my mono-repository on GitHub.

## Future

Create a Python package which packages up the modules for creating users. To
date, I have simply copied the modules into the project where I needed them.

[1]: https://data.sa.gov.au/data/dataset/popular-baby-names
[2]: https://data.nsw.gov.au/data/dataset/deceased-estate-files-1880-1923
[3]: https://www.abs.gov.au/AUSSTATS/abs@.nsf/DetailsPage/1270.0.55.003July%202016?OpenDocument
[name_people.py]: https://github.com/donno/warehouse51/blob/d551cc46bdf412110b7983b62a22738851c6a47d/snippets/name_people.py
[name_company.py]: https://github.com/donno/warehouse51/blob/d551cc46bdf412110b7983b62a22738851c6a47d/snippets/name_company.py
[user_generator.py]: https://github.com/donno/warehouse51/blob/d551cc46bdf412110b7983b62a22738851c6a47d/snippets/user_generator.py
