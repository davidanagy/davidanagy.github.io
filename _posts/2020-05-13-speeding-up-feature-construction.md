---
layout: post
title: Speeding Up Feature Construction
---
(NOTE: This article assumes basic knowledge of Python and pandas. See pandas documentation [here](https://pandas.pydata.org/).)

In my [previous article](https://davidanagy.github.io/2020-04-08-automatically-scraping-and-updating-data-in-flask/), I discussed the "Sanctions Explorer" project I worked on for two months and one of the main challenges I faced: how to automatically scrape new data and update the database with it. This article is about another main challenge: how to construct a particular sort of new feature in a reasonable timeframe. First, I will explain [the overall problem](#stating-the-problem) I was faced with, including my first inadequate solution. Then I will discuss [my final solution](#sorting-to-the-rescue!).

### Stating the Problem

In the article linked above, I stated that finding information on *historical* sanctions--i.e., entities that used to be, but are no longer, under sanction--is currently very difficult, as this information is only contained in long .txt files published by the US Treasury Department. Our client, the [Center for Advanced Defense Studies](https://c4ads.org/) (hereafter "C4ADS"), provided us a parser that took this raw text data and transformed it into a structured table. But there was a problem: due to the way the Trreasury Department structures their text data, multiple distinct rows in this dataset actually belonged to the *same entry*. I'll explain what I mean with an example.

Let's say the US decides to sanction Fred Smith on January 1, 2001. In their text file, this is recorded by listing Fred Smith as one of the entries that was "added" on 1/1/01; in C4ADS's data, this is logged as a row with `added` in an `action` column, with `1/1/01` in a `date` column. Then, on 2/2/02, they discover a new address of Fred Smith's and add this address to his file. Whenever a change occurs, the text data lists the old entry **followed by** the new entry; C4ADS's data records this as two rows, one with `changed_from` in the `action` column, the other with `changed_to` in that column. Finally, on 3/3/03, they de-list Fred Smith (i.e., no longer sanction him); this too is recorded as a new row, with `removed` in the `action` column. All told, Fred Smith has four total rows in the dataset, even though he's only one person.

(It was actually somewhat more complicated than this since the data had some errors [inevitable, given the extremely varied nature of the original text data]. But while dealing with these errors did take a great deal of time and creativity, it's not quite the same issue I'm addressing in this article, so I won't talk about it in this essay.)

This, then, was the problem, which I described in my previous article as "figure out which rows were related to which other rows." The client wanted us to list all previous changes to an entry when users searched for it. Given that all I had was the information about each individual or entity, the `action` that was committed, and the `date` that action was performed on, how could I accurately link these entries together?

I decided that the most straightforward way to link the entries would be to create a new column called `entry_id_no`. This would contain index labels (i.e. whole numbers starting at 0), except that any two rows that belonged to the same entry would have the same Entry ID Number. That way, when the user searches, we would be able to group together rows with the same Entry ID Number so that the web frontend could display all the information together. So the next question was, how to make these Entry ID Numbers? My first idea was the following:

1. Iterate through each row.

2. If a row has `added` in the `action` column, assign it a new Entry ID Number (since a newly-added entry is presumably separate from all previous entries).

3. Otherwise, look through each other row to find an *identical* row (minus certain fields I know will be different, such as `action` or `date`) that already has an Entry ID Number. Once we do, assign it the identical row's Entry ID Number.

The reader may have already noticed the problem here: for each row, unless that row says `added`, I'm running a *nested* loop that goes through all other rows. In other words, the [time complexity](https://en.wikipedia.org/wiki/Time_complexity) of this method is [O(n<sup>2</sup>)](https://en.wikipedia.org/wiki/Big_O_notation), rendering it unsuitable for production use. (Indeed, I never let the code run in its entirety, since it was estimating to take at least 17 hours.)

This, then, was the challenge: how could I link the entries together in a reasonable timeframe?

### Sorting to the Rescue!

First, I could reduce some time by making use of the `changed_to` action. Recall that when an entry is changed, *two* additional rows are added: a `changed_from` row that contains the entry's original information, and a `changed_to` row that contains the new, changed entry. In other words, **only** the `changed_from` row should be identical to a previous row; the `changed_to` row, on the other hand, always comes **directly after** the `changed_from` row it's paired with. So I separated out the `changed_to` rows at the beginning of my loop, and then at the end, merely had to give each of them the Entry ID Number I had assigned to the `changed_from` row directly above it. The time complexity of this latter operation was just O(n), a significant improvement.

That was all well and good--but I was still faced with O(n<sup>2</sup>) time complexity for the `changed_from` and the `removed` rows, as years or even decades could theoretically pass between the original `added` row and its change or deletion. I needed some way to shorten the amount of time it took to find a duplicate row. As you might be able to guess from the title to this section, I accomplished this by [*sorting* the data](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.sort_values.html).

Now, this idea itself is nothing original; I'm **searching** for a duplicate row, and a fast [binary search](https://www.khanacademy.org/computing/computer-science/algorithms/binary-search/a/binary-search) always requires sorting first. But the process I went through is quite a bit more complicated than a simple sort and search. The data we possessed contained almost 100 features, and we had to ensure **all** of them (save, again, for the few I knew would be different such as `date` and `action`) were identical. Moreover, every column had a large percentage of null values, and so sorting just one column didn't help much because the nulls all got clumped together.

To address the latter issue, I decided to sort based on multiple columns at once. The `last_name` and `entity_name` columns both had a significant number of actual values. Brief analysis showed that when one had a null, the other tended to have an actual name--presumably, "individuals" (i.e., persons) had a "last name," while "entities" (i.e. organizations, vessels, and aircraft) had an "entity name." So by sorting first on `entity_name` and then on `last_name`, the vast majority of rows were put in a legitimate order. (To sort by multiple columns in pandas, simply put a list of column names after the `by` parameter of the `sort_values` method.)

Next was the trickiest part--figuring out the right way to limit the search based on this sort. I decided to use a feature of the data to my advantage: the fact that it's **chronological**. Theoretically, each entry should be `added` first, and only after that changed, removed, etc. Furthermore, conveniently, the chronological order of each action is equivalent to its alphabetical order: `added` --> `changed_from` --> `changed_to` --> `removed`. Thus, along with sorting by `last_name` and `entity_name`, I also sorted by `date` and then `action`. This meant that each "set" of rows with identical last/entity names were themselves sorted chronologically.

(Technically, I [converted the date column to datetime format](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.to_datetime.html), then split it into separate `year`, `month`, and `day` columns before sorting by each of those in order, year --> month --> day. This was to ensure that the sort worked correctly.)

The following step was the key one. I proceeded to make a list of "value count dictionaries"--two dictionaries to be precise, one for `entity_name`, the other for `last_name`. For each of those columns, I called on [pandas's value_counts method](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.Series.value_counts.html). `value_counts` outputs a pandas Series, with the index being each unique word in the column and the values being the number of times that word appears in the column. The "value count dictionaries" are simply dictionaries with those index words as keys and the values as, well, values. Here's the code itself:

    val_count_dicts = []
    # For each column in the sort_by list,
    for col_name in sort_by:
        # Run the value_counts method on that column (including NaN values).
        # Define the result as "val_count."
        val_count = df4[col_name].value_counts(dropna=False)
        # Start with an empty dictionary.
        val_count_dict = {}
        # Run a for-loop for each string in val_count.
        for i in range(len(val_count)):
            # val_count.index contains each unique string in the given sort_by column.
            # So we set a key equal to the string in position i in the index,
            # and set the value to be the number in position i in val_count;
            # i.e., the number of times that string appears in the column.
            # This constructs the "value count dictionary" I explained above.
            val_count_dict[val_count.index[i]] = val_count[i]
        # Append the completed value count dictionary to the val_count_dicts list.
        val_count_dicts.append(val_count_dict)

Why did I go through all this trouble? Because the val_count_dicts show us the **maximum number of rows** we have to search through to find a duplicate row. For instance, let's say we're looking at Fred Smith's `changed_from` row. If we know there are only 10 "Smiths" total, we only have to look through 10 other rows for a duplicate--after all, no other row can possibly have "Smith" in the `last_name` column. And since the data has been sorted chronologically after we sorted by name, that means we only have to look at the **previous** 10 rows, since one of those 10 must necessarily be the relevant `added` or `changed_to` one.

Finally, the only issue left was determining which `val_count_dict` to consult for each row. But this was simple--for each row, I detected which column has a non-null value in it, and then used the relevant `val_count_dict`. (If both columns were null, I defaulted to the `entity_name` val_count_dict.) Then it was just a matter of plugging the value itself into the `val_count_dict`, getting back the appropriate count, and looking through that many previous rows (up to the beginning of the dataset).

All told, while this method may be much more difficult to understand for humans than my earlier, O(n<sup>2</sup>) one, for a computer it was much less complex to handle--only O(n log n). Instead of taking over 16 hours to complete, this process took my computer less than 10 minutes. As is often the case, a human putting in a lot of extra time at the beginning saves the computer far more time in the end!

Please feel free to email me ([davidanagy@gmail.com](mailto:davidanagy@gmail.com)) with any questions or concerns. (You can also contact me via social media; links below.) Thanks for reading!