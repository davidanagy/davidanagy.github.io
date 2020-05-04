---
layout: post
title: Automatically Scraping and Updating Data in Flask
---
(NOTE: This article assumes basic knowledge of Python and Flask. See Flask documentation [here](https://flask.palletsprojects.com/en/1.1.x/).)

For two months, I worked with a team to help make historical sanctions data more easily accessible and searchable; see my colleague Max Efremov's [article here](https://maxefremov.com/sanctionsexplorer/) for more details on the project and its importance. While there are many current websites that enable one to search through (at least US) sanctions data, they all only contain *current* data on entities that are *at present* sanctioned; it is much more difficult to find information about entities that *used to be* (but are no longer) under sanction. It it this historical sanctions data that we focused on integrating into our website.

The US Treasury Department has text files containing all changes made to their sanctions policies. These files are available on their website [here](https://www.treasury.gov/resource-center/sanctions/SDN-List/Pages/archive.aspx), from 1994 to the present day. Our sponsor, the [Center for Advanced Defense Studies](https://c4ads.org/), provided us with a parser that takes these raw text files and puts their relevant information into a structured table. Our job was to take this tabular data and (a) make it searchable; (b) figure out which rows were related to which other rows; (c) automatically scrape new Treasury data and add it to our database; and more. This article will focus on how we accomplished that last task, which was assigned to me: scrape new Treasury data, run it through the parser, and add it to our database. I will discuss how I accomplished this, focusing on three major challenges:

1. [Extracting only the new data from the Treasury .txt file](#extracting-new-data-only);

2. [Running the scraper in Flask automatically](#automatic-scraping-in-flask);

3. [Making the app update along with the database](#updating-the-app).

### Extracting New Data Only

If you look at an example .txt file, such as [the 2020 one](https://www.treasury.gov/ofac/downloads/sdnnew20.txt), you will see that the data in it is divided by date: first it states the date (e.g. "01/03/20:"), followed by a list of changes made to the sanctions data made on that date--added entries, changes to previous entries, entries being removed, etc. Then it states a new date (e.g. "01/08/20:"), a list of changes made on *that* date, and so on. To successfully scrape this data and add it to our dataset, then, I needed to accomplish the following:

a) Download the .txt file corresponding to the **current year** (if I hardcoded in 2020, then the code would have to be updated every year).

b) Determine if any changes had been made since the previous scrape.

c) If there are changes, determine just what those changes are.

d) Use the parser provided by the stakeholder to put the new data in tabular format.

(e) Add the new tabular data to our dataset.

(f) Clean up our files.

(a) was relatively straightforward. Thankfully, Treasury has relatively consistent naming conventions; the URLs (at least recently) just differ by year. I just had to get the current year by importing `date` from Python's inbuilt `datetime` library and running `date.today().year`. After transforming it into a string and pulling out the last two digits (since that's how Treasury differentiates years), I saved the new two-character string (e.g. `'20'`) as `url_year`, and got the relevant URL using the [f-string](https://realpython.com/python-f-strings/) `f'https://www.treasury.gov/ofac/downloads/sdnnew{url_year}.txt'`. Finally, after importing the inbuilt `ulrlib.request` module, I just used the `urllib.request.urlretrieve()` function to obtain my file.

There are two problems with my solution. First, it's possible that Treasury may someday change their URL naming conventions, in which case my URL would no longer work. But it would be very difficult or impossible to fully control for that; it's just something the proprietors of the website code need to keep an eye on. Second, the `urllib.request.urlretrieve()` function [may become deprecated](https://docs.python.org/3/library/urllib.request.html) at some point in the future, but I couldn't find a different function that accomplishes the same goal as easily and efficiently, so for the time being at least, I think it's the best solution--at least until the function actually **does** get deprecated (if ever).

(b), determining if changes had been made since the previous scrape, was also relatively straightforward. First, I had to keep around a copy of the current year's .txt file from the previous scrape, which I called `sdn_old.txt` ("SDN" because this is specifically SDN or "[Specially Designated Nationals and Blocked Persons](https://www.treasury.gov/resource-center/sanctions/sdn-list/pages/default.aspx)" data). Then, after saving the newly-scraped file as `sdn_new.txt`, I simply read them both as strings in Python and compared the length of each. If they had the same length, I assumed that meant they were the same file, so I just deleted the newly-downloaded `sdn_new.txt`. If they had different lengths, that meant they must be different files, so I moved on to the next step.

Step (c) was probably the most complicated one. Even if the newly-downloaded `sdn_new.txt` is different from the `sdn_old.txt` currently in our files, much or most of it was going to be the same, since Treasury just adds new changes to the end of the pre-existing file. I only wanted to parse the data that was different, so I needed to determine exactly what those changes were. Luckily, the Treasury department's .txt files are a sort of time series; since the changes are listed in time order, from earliest to latest, that means that the new data must be at the **end** of the file. In other words, what I really needed to figure out was at which point *the changes began*; I could assume that once I found that, all the data after that point would be new.

After analyzing the text documents, I decided the best way to do this would be by searching for the last two digits of the year (called `url_year` above) followed by a colon, e.g., `'20:'`, as that string only appeared in the date that's listed before each set of changes made on that date (as I described earlier). I could count how many times this string appears in `sdn_old.txt`--call this number "n." Then, I'd just need to find the *nth plus one* time that it appears in `sdn_new.txt`. (For example, if "20:" appears 10 times in `sdn_old.txt`, I need to find the 11th time it appears in `sdn_new.txt`.) This location must be where the new date(s) begin in `sdn_new.txt`, so I could just take everything in the document from that point and run it through the parser. After finding a [useful geeks2geeks](https://www.geeksforgeeks.org/python-ways-to-find-nth-occurrence-of-substring-in-a-string/) article about finding the nth occurrence of a substring in a string, I used their second method to successfully pull out the new, changed data. Finally, I saved this as a .txt file, since the parser provided to us worked specifically by parsing .txt files.

There is one special case that had to be considered here: what if we had just entered a new year, so that `sdn_new` contains (e.g.) 2021 data while `sdn_old` has 2020 data? To account for this possibility, I wrote a conditional statement: if `f'{url_year}'` (e.g. "21:") doesn't appear at all in `sdn_old`, that means that `sdn_new` contains **entirely** new data (as it's of a different year from `sdn_old`), so I parsed the entire file.

I can't go into too much detail on step (d), parsing the new data, since the parser our stakeholder provided us is proprietary. It wouldn't be of much help to you anyway since how precisely to apply a parser to raw text data differs from parser to parser. This step resulted in a new CSV file, which I then [read into a pandas dataframe](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.read_csv.html#pandas.read_csv).

Step (e), adding the newly-parsed tabular data to our dataset, was a bit challenging only because we added several columns to the parsed, tabular data for our own purposes, so I needed to make sure to correctly add/create the same columns before using the pandas [concat method](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.concat.html) to add the new data to the end of the main dataset.

Finally, step (f), cleaning up our files, is necessary because this process creates several files that are no longer necessary after the scraping is finished; namely, `sdn_old.txt`, the .txt file I made with only the new data, and the CSV file containing only the new data. I used the builtin `os` library's `remove()` function to do this. Then I renamed `sdn_new.txt` to `sdn_old.txt` (since, after all, it is now the "old" text file that I want to compare future scraped files against) using `os.rename()`, and saved the new combined dataframe as a CSV file with pandas's [to_csv() method](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.to_csv.html). And that was it--the data had been successfully scraped, parsed, and added. Now I needed to figure out how to get this to happen automatically.

### Automatic Scraping in Flask

The data science product was effectively a backend to the frontend website (which was the web team's product), programmed in Flask. It would obviously be un-ideal to have to manually go through the above process each time we wanted to update our database. Hence, it was necessary for me to figure out a way to get the above code to run in our Flask app automatically.

After some research, I determined that the [Flask-APScheduler library](https://github.com/viniciuschiele/flask-apscheduler), which takes the [Advanced Python Scheduler](https://apscheduler.readthedocs.io/en/stable/) (or "APScheduler") library and makes it easier to use in Flask, was best for this. While it does not have fantastic documentation, the github repository linked above does contain several examples, and I used [this one](https://github.com/viniciuschiele/flask-apscheduler/blob/master/examples/jobs.py) as a reference for my own code.

First, in my main `application.py` file, after a `from flask_apscheduler import APScheduler` line, I defined a new class as such:

	class Config(object):
    	JOBS = [
        	{
            	'id': 'ofac_scraper',
            	'func': 'scraping:scrape_ofac_data',
            	'trigger': 'interval',
            	'days': 1
        	}
    	]

    	SCHEDULER_API_ENABLED = True

Despite being a class, it has no constructor; instead, it just has two variables, `JOBS` and `SCHEDULER_API_ENABLED`. The latter is set to `True`, but the main action is in the former. `JOBS` is defined as a list of dictionaries, each dictionary corresponding to one action, or job, we want to perform automatically. In this case, I only had one such job--run the scraping algorithm as detailed above--so I only included one dictionary.

The `id` can be anything you want, but it should obviously be descriptive. The `trigger` depends on what kind of schedule we want the job to run on (see the [main APScheduler docs](https://apscheduler.readthedocs.io/en/stable/userguide.html) for more information); since I want to scrape automatically at set intervals, I set it to `interval`. With an interval trigger, you also need to define the interval. Because Treasury doesn't update their data that often, I decided once every day was sufficient so I set a `days` key equal to `1`. For testing, though, I shortened this significantly to `'minutes': 5`. Since the scraping code takes a few minutes to run (due mainly to the new columns I added in step (e)), problems arose if I made it any shorter, since then the job would be in the middle of executing before running again. Finally, the `func` key tells the scheduler which function we want it to run. First, you write the name of the file that contains the function (in my case, it was `scraping.py`), then a colon, and then the name of the function itself.

After defining this new `Config` class, you then have to make sure the scheduler itself runs when the application starts. At the bottom of `application.py`, after the `if __name__ == "__main__":` but before the final `application.run()` command, enter the following lines (or equivalents, if your application variable is something other than `application`):

	application.config.from_object(Config())
	
	scheduler = APScheduler()
	scheduler.init_app(application)
	scheduler.start()

And that's it! While I do find the code itself here rather unintuitive, it was relatively straightforward once I figured out how it worked.

There was one final step I had to complete, though. As stated above, the result of the scraping is a new CSV file with the updated data. However, just because the CSV updates does not mean that the *application itself* updates with the new data. I will explain why in the section below.

### Updating the App

To access the data stored in the CSV, we programmed it such that when the Flask app first boots up, it reads the CSV file into a Pandas dataframe which we saved under a `df` variable. We then modified `df`, searched through it, etc. The problem was that, even though the underlying CSV may change, the *dataframe saved in memory*--df--stays the same (at least, until the app reboots and runs all its code again). So the final challenge was to find a way to update `df` along with the underlying CSV file.

You might wonder what the problem here is--can't I just re-define `df` automatically after running the scraper? But it wasn't that simple. APScheduler needs the job to be defined inside a function--`scrape_ofac_data` in our case--while I need to update `df` *globally*. Furthermore, to keep things (relatively) clean, this function was in a totally separate file (`scraping.py`) from out main application file (`application.py`), and the latter is the one that needs access to the updated `df`. It took a fair bit of research and trial and error, but I eventually found a way to solve these problems. [This article](https://www.programiz.com/python-programming/global-keyword) in particular was quite helpful.

The first problem was the easier one. By defining `df` (as well as various other variables `application.py` needs access to) as global *within* `scrape_ofac_data`, we can make changes to it within the function (that gets run automatically) and have those changes be reflected on the global level:

	global df

However, to do this, `df` needs to be defined within `scraping.py` *and not* `application.py`. How does the latter gain access to it? By *importing scraping* within `application.py`:

	import scraping

Then, since `df` is defined within `scraping`, we can just call `scraping.df` to access it.

This is the key, though: for `df` to update appropriately, we have to import *the entire scraping file*. In other words, when I tried `from scraping import df`, the `df` variable stayed the same even after the scraping code was run. It appears that the only way to share a single global variable among multiple files--`df` in this case--is to import the entire file in which it's defined.

### Conclusion

In order to successfully scrape new OFAC data and add it to our database, I had to go through three steps, each of which posed its own unique, complicated problems. By the end, though, not only had I succeeded in making the product, I also learned much about Python, Flask, and problem-solving in general. I hope this article helped you as well!

Please feel free to comment or email me ([davidanagy@gmail.com](mailto:davidanagy@gmaill.com)) with any questions or concerns. Thanks for reading!