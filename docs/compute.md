New columns: Using .compute()
=============================

This provides some examples of the [.compute() method](http://agate.readthedocs.io/en/1.6.0/cookbook/compute.html), which allows you to create new columns based on a value already in the data.

## zfill: Fix zero-padded IDs

In school data from TEA, the `Campus_ID` column is supposed to be a 9-digit column, and any school that is less than that should have zeros at the beginning. Sometimes .csv files saved from Excel loose this zero padding, but we can replace it:

```
  zfilled = nodistrict.compute([
      ('New_Campus_ID', agate.Formula(agate.Text(), lambda r: r['CAMPUS'].zfill(9))),
      ('New_District_ID', agate.Formula(agate.Text(), lambda r: r['DISTRICT'].zfill(6)))
  ])
```

For the first one, we created `New_Campus_ID` by going through the `Campus` column and using the Python method `.zfill()` to set pad any `Campus` that wasn't 9 characters. For  `['District']` we used "6".

## Mapped translation

We wanted to create a new field that inserts the longer explanation based on the a one-letter designation in the data.

First we have the map:

``` python
  rating_map = {
      'I': 'Improvement required',
      'M': 'Met standard',
      'A': 'Met alternative standard',
      'X': 'Not rated',
      'Z': 'Not rated',
      '': 'Not rated'
  }
```

Then we need a function that we will run the compute through to get the proper rating match:

``` python
  def map_rating(rating):
      rating = rating.strip()
      return rating_map[rating]
```

Then we create the new table. We are looking at the field ``C_RATING`` for the single-letter values. (This was originally used as a method on an export command, so it should be tested in this configuration):

``` python
  rated = unrated.compute([
      ('mapped_rating',
       agate.Formula(agate.Text(),
       lambda r: map_rating(r['C_RATING']))
      )
  ])
```

In then end, we have a new column called ``mapped_rating`` that includes values like "Met Standard".

## Converting timezones

I had data that came in UTC time, but I wanted to display it in Central Time. You first need to make sure your datatime is not naive and has a timezone, perhaps when [you import it](#add-timezone-to-a-date).

In this case, I had a field `dateTime` that was in UTC, that I need to convert to Central Time. This requires  the [pytz library](http://pytz.sourceforge.net/index.html?highlight=list%20timezones#), as well:

``` python
  ## Import pytz if you don't already have it
  import pytz

  ## setting central time
  central = pytz.timezone('US/Central')

  ## formula to do the conversion from the 'dateTime' field
  time_shifter = agate.Formula(
    agate.DateTime(),
    lambda r: r['dateTime'].astimezone(central)
    )

  ## create column and call the formula above
  flow_central = flow_data.compute([
          ('Central Time', time_shifter),
      ])
```

This gives me the new column `Central Time`.

## Rewrite date/time in another format


I had a case where Tableau did not understand the "native" datetime format (2017-08-26 22:00:00-05:00) that was exported to csv, so I had to create a new "pretty" date column (2017-08-26 22:00:00). I had lots of challenges because I could not format both a time and date `.stftime()` from the `.date()` method, nor the `.time()` method. It would only understand it's own type. So, I created strings of the date and the time and then put them together. Since I was exporting, it didn't matter that it was text and not a true datetime object.:

``` python
  # create a tableau-friendly date
  flow_central = flow_central.compute([
          ('Measurement time', agate.Formula(agate.Text(),
                                       lambda r: str(r['CentralTime'].date())\
                                       + " " \
                                       + str(r['CentralTime'].time()))) 
      ])
```

## Reshaping table to collect selected columns

This is a interesting case I've had a couple of times with education data. Basically the data set starts as a basic table where each column has a separate value, but they are related in some way.  You might have a number and a percentage for each race: `White total`, `White percent`, `Black total`, `Black percent`, `Hispanic total`, `Hispanic percent`.  We want to turn those into two three columns: `Race`, `Total`, `Percentage`.

There is probably a better way to do this than the way I have tackled it here. The process I'm using goes like this: 
- I compute specific pipe-delimited combo columnns with related data.
- I create a table with just the columns we need
- Normalize the data to get a row for each combo columns
- Split the combo columns into individual columns
- Reset the column names


