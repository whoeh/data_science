.. title: Extracting Dates From Medical Data
.. slug: extracting-dates-from-medical-data
.. date: 2017-08-07 17:16
.. tags: regularexpressions
.. link: 
.. description: This assigniment uses regular expressions to extract dates from medical data and sorts them.
.. type: text
.. author: necromuralist


1 Introduction
--------------

In this assignment, you'll be working with messy medical data and using regular expressions to extract relevant information from the data. 

Each line of the ``dates.txt`` file corresponds to a medical note. Each note has a date that needs to be extracted, but each date is encoded in one of many formats.

The goal of this assignment is to correctly identify all of the different date variants encoded in this dataset and to properly normalize and sort the dates. 

Here is a list of some of the variants you might encounter in this dataset:

- 04/20/2009; 04/20/09; 4/20/09; 4/3/09

- Mar-20-2009; Mar 20, 2009; March 20, 2009;  Mar. 20, 2009; Mar 20 2009;

- 20 Mar 2009; 20 March 2009; 20 Mar. 2009; 20 March, 2009

- Mar 20th, 2009; Mar 21st, 2009; Mar 22nd, 2009

- Feb 2009; Sep 2009; Oct 2010

- 6/2008; 12/2009

- 2009; 2010

Once you have extracted these date patterns from the text, the next step is to sort them in ascending chronological order accoring to the following rules:

- Assume all dates in xx/xx/xx format are mm/dd/yy

- Assume all dates where year is encoded in only two digits are years from the 1900's (e.g. 1/5/89 is January 5th, 1989)

- If the day is missing (e.g. 9/2009), assume it is the first day of the month (e.g. September 1, 2009).

- If the month is missing (e.g. 2010), assume it is the first of January of that year (e.g. January 1, 2010).

With these rules in mind, find the correct date in each note and return a pandas Series in chronological order of the original Series' indices.

For example if the original series was this:

::

    0    1999
    1    2010
    2    1978
    3    2015
    4    1985

::

    0    2
    1    4
    2    0
    3    1
    4    3

Your score will be calculated using `Kendall's tau <https://en.wikipedia.org/wiki/Kendall_rank_correlation_coefficient>`_, a correlation measure for ordinal data.

**This function should return a Series of length 500 and dtype int.**

2 Imports
---------

.. code:: ipython

    # from pypi
    import pandas

3 Loading The Data
------------------

.. code:: ipython

    with open('dates.txt') as reader:
        data = pandas.Series(reader.readlines())

    data.head(10)

::

    0         03/25/93 Total time of visit (in minutes):\n
    1                       6/18/85 Primary Care Doctor:\n
    2    sshe plans to move as of 7/8/71 In-Home Servic...
    3                7 on 9/27/75 Audit C Score Current:\n
    4    2/6/96 sleep studyPain Treatment Pain Level (N...
    5                    .Per 7/06/79 Movement D/O note:\n
    6    4, 5/18/78 Patient's thoughts about current su...
    7    10/24/89 CPT Code: 90801 - Psychiatric Diagnos...
    8                         3/7/86 SOS-10 Total Score:\n
    9             (4/10/71)Score-1Audit C Score Current:\n
    dtype: object

.. code:: ipython

    data.describe()

::

    count                                                   500
    unique                                                  500
    top       sApproximately 7 psychiatric hospitalizations ...
    freq                                                      1
    dtype: object

4 The Grammar
-------------

4.1 Cardinality
~~~~~~~~~~~~~~~

.. code:: ipython

    ZERO_OR_MORE = '*'
    ONE_OR_MORE = "+"
    ZERO_OR_ONE = '?'
    EXACTLY_TWO = "{2}"
    ONE_OR_TWO = "{1,2}"
    EXACTLY_ONE = '{1}'

4.2 Groups and Classes
~~~~~~~~~~~~~~~~~~~~~~

.. code:: ipython

    GROUP = r"({})"
    NAMED = r"(?P<{}>{})"
    CLASS = "[{}]"
    NEGATIVE_LOOKAHEAD = "(?!{})"
    NEGATIVE_LOOKBEHIND = "(?<!{})"
    POSITIVE_LOOKAHEAD = "(?={})"
    POSITIVE_LOOKBEHIND = "(?<={})"
    ESCAPE = "\{}"

4.3 Numbers
~~~~~~~~~~~

.. code:: ipython

    DIGIT = r"\d"
    ONE_DIGIT  = DIGIT + EXACTLY_ONE
    ONE_OR_TWO_DIGITS = DIGIT + ONE_OR_TWO
    NON_DIGIT = NEGATIVE_LOOKAHEAD.format(DIGIT)
    TWO_DIGITS = DIGIT + EXACTLY_TWO
    THREE_DIGITS = DIGIT + "{3}"
    EXACTLY_TWO_DIGITS = DIGIT + EXACTLY_TWO + NON_DIGIT
    FOUR_DIGITS = DIGIT + r"{4}" + NON_DIGIT

4.4 String Literals
~~~~~~~~~~~~~~~~~~~

.. code:: ipython

    SLASH = r"/"
    OR = r'|'
    LOWER_CASE = "a-z"
    SPACE = "\s"
    DOT = "."
    DASH = "-"
    COMMA = ","
    PUNCTUATION = CLASS.format(DOT + COMMA + DASH)
    EMPTY_STRING = ""

4.5 Dates
~~~~~~~~~

These are parts to build up the date-expressions.

.. code:: ipython

    MONTH_SUFFIX = (CLASS.format(LOWER_CASE) + ZERO_OR_MORE
                    + CLASS.format(SPACE + DOT + COMMA + DASH) + ONE_OR_TWO)
    MONTH_PREFIXES = "Jan Feb Mar Apr May Jun Jul Aug Sep Oct Nov Dec".split()
    MONTHS = [month + MONTH_SUFFIX for month in MONTH_PREFIXES]
    MONTHS = GROUP.format(OR.join(MONTHS))
    DAY_SUFFIX = CLASS.format(DASH + COMMA + SPACE) + ONE_OR_TWO
    DAYS = ONE_OR_TWO_DIGITS + DAY_SUFFIX
    YEAR = FOUR_DIGITS

This is for dates like *Mar 21st, 2009*, those with suffixes on the days.

.. code:: ipython

    CONTRACTED = (ONE_OR_TWO_DIGITS
                  + LOWER_CASE
                  + EXACTLY_TWO
                  )
    CONTRACTION = NAMED.format("contraction",
                               MONTHS
                               + CONTRACTED
                               + DAY_SUFFIX
                               + YEAR)

This is for dates that have no days in them, like *May 2009*.

.. code:: ipython

    NO_DAY_BEHIND = NEGATIVE_LOOKBEHIND.format(DIGIT + SPACE)
    NO_DAY = NAMED.format("no_day", NO_DAY_BEHIND + MONTHS + YEAR)

This is for the most common form (that I use) - *May 21, 2017*.

.. code:: ipython

    WORDS = NAMED.format("words", MONTHS + DAYS + YEAR)

This is for the case where the day is placed before them month - *20 March, 2009*.

.. code:: ipython

    BACKWARDS = NAMED.format("backwards", ONE_OR_TWO_DIGITS + SPACE + MONTHS + YEAR)

This is the case where slashes are used but only two digits were used for the year (so we're assuming it's in the twentieth century) - *8/4/98*.

.. code:: ipython

    slashed = SLASH.join([ONE_OR_TWO_DIGITS,
                          ONE_OR_TWO_DIGITS,
                          EXACTLY_TWO_DIGITS])
    dashed = DASH.join([ONE_OR_TWO_DIGITS,
                        ONE_OR_TWO_DIGITS,
                        EXACTLY_TWO_DIGITS])
    TWENTIETH_CENTURY = NAMED.format("twentieth",
                                     OR.join([slashed, dashed]))

This is the case where digits with slashes are used and all four digits are used for the year - *8/4/1998*.

.. code:: ipython

    NUMERIC = NAMED.format("numeric",
                           SLASH.join([ONE_OR_TWO_DIGITS,
                                       ONE_OR_TWO_DIGITS,
                                       FOUR_DIGITS]))

This is the case where only month and year are given as digits - *9/2009*. There are two expressions, because the day can be one or two digits.

.. code:: ipython

    NO_PRECEDING_SLASH = NEGATIVE_LOOKBEHIND.format(SLASH)
    NO_PRECEDING_SLASH_DIGIT = NEGATIVE_LOOKBEHIND.format(CLASS.format(SLASH + DIGIT))
    NO_ONE_DAY = (NO_PRECEDING_SLASH_DIGIT
                  + ONE_DIGIT
                  + SLASH
                  + FOUR_DIGITS)
    NO_TWO_DAYS = (NO_PRECEDING_SLASH
                   + TWO_DIGITS
                   + SLASH
                   + FOUR_DIGITS)
    NO_DAY_NUMERIC = NAMED.format("no_day_numeric",
                                  NO_ONE_DAY
                                  + OR
                                  + NO_TWO_DAYS
                                  )

This is the case where only a year was given. This is the hardest case, since you don't want to accidentally match the other cases, but the text preceding and following it could be anything. For the look-behind, all the cases have to have the same number of characters so we can't re-use the other expressions

.. code:: ipython

    CENTURY = GROUP.format('19' + OR + "20") + TWO_DIGITS
    DIGIT_SLASH = DIGIT + SLASH
    DIGIT_DASH = DIGIT + DASH
    DIGIT_SPACE = DIGIT + SPACE
    LETTER_SPACE = CLASS.format(LOWER_CASE) + SPACE
    COMMA_SPACE = COMMA + SPACE
    YEAR_PREFIX = NEGATIVE_LOOKBEHIND.format(OR.join([
        DIGIT_SLASH,
        DIGIT_DASH,
        DIGIT_SPACE,
        LETTER_SPACE,
        COMMA_SPACE,    
    ]))

    YEAR_ONLY = NAMED.format("year_only",
                             YEAR_PREFIX + CENTURY
    )

These are leftovers that don't really match anything.

.. code:: ipython

    IN_PREFIX = POSITIVE_LOOKBEHIND.format(CLASS.format('iI') + 'n' + SPACE) + CENTURY
    SINCE_PREFIX = POSITIVE_LOOKBEHIND.format(CLASS.format("Ss") + 'ince' + SPACE) + CENTURY
    AGE = POSITIVE_LOOKBEHIND.format("Age" + SPACE + TWO_DIGITS + COMMA + SPACE) + CENTURY
    AGE_COMMA = POSITIVE_LOOKBEHIND.format("Age" + COMMA + SPACE + TWO_DIGITS + COMMA + SPACE) + CENTURY
    OTHERS = ['delivery', "quit", "attempt", "nephrectomy", THREE_DIGITS]
    OTHERS = [POSITIVE_LOOKBEHIND.format(label + SPACE) + CENTURY for label in OTHERS]
    OTHERS = OR.join(OTHERS)
    LEFTOVERS_PREFIX = OR.join([IN_PREFIX, SINCE_PREFIX, AGE, AGE_COMMA]) + OR + OTHERS
    LEFTOVERS = NAMED.format("leftovers", LEFTOVERS_PREFIX)

This is the combined expression for all the dates - the one that should be used to extract them from the data.

.. code:: ipython

    DATE = NAMED.format("date", OR.join([NUMERIC,
                                         TWENTIETH_CENTURY,
                                         WORDS,
                                         BACKWARDS,
                                         CONTRACTION,
                                         NO_DAY,
                                         NO_DAY_NUMERIC,
                                         YEAR_ONLY,
                                         LEFTOVERS]))

.. code:: ipython

    def twentieth_century(date):
        """adds a 19 to the year

        Args:
         date (re.Regex): Extracted date
        """
        month, day, year = date.group(1).split(SLASH)
        year = "19{}".format(year)
        return SLASH.join([month, day, year])

.. code:: ipython

    def take_two(line):
        match = re.search(TWENTIETH_CENTURY, line)
        if match:
            return twentieth_century(match)
        return line

5 Applying The Grammer
----------------------

.. code:: ipython

    def extract_and_count(expression, data, name):
        """extract all matches and report the count

        Args:
         expression (str): regular expression to match
         data (pandas.Series): data with dates to extratc
         name (str): name of the group for the expression

        Returns:
         tuple (pandas.Series, int): extracted dates, count
        """
        extracted = data.str.extractall(expression)[name]
        count = len(extracted)
        print("'{}' matched {} rows".format(name, count))
        return extracted, count

.. code:: ipython

    numeric, numeric_count = extract_and_count(NUMERIC, data, 'numeric')

::

    'numeric' matched 25 rows

.. code:: ipython

    twentieth, twentieth_count = extract_and_count(TWENTIETH_CENTURY, data, 'twentieth')

::

    'twentieth' matched 100 rows

.. code:: ipython

    words, words_count = extract_and_count(WORDS, data, 'words')

::

    'words' matched 34 rows

.. code:: ipython

    backwards, backwards_count = extract_and_count(BACKWARDS, data, 'backwards')

::

    'backwards' matched 69 rows

.. code:: ipython

    contraction_data, contraction = extract_and_count(CONTRACTION, data, 'contraction')

::

    'contraction' matched 0 rows

.. code:: ipython

    no_day, no_day_count = extract_and_count(NO_DAY, data, 'no_day')

::

    'no_day' matched 115 rows

.. code:: ipython

    no_day_numeric, no_day_numeric_count = extract_and_count(NO_DAY_NUMERIC, data,
                                                             "no_day_numeric")

::

    'no_day_numeric' matched 112 rows

.. code:: ipython

    year_only, year_only_count = extract_and_count(YEAR_ONLY, data, "year_only")

::

    'year_only' matched 15 rows

.. code:: ipython

    leftovers, leftovers_count = extract_and_count(LEFTOVERS, data, "leftovers")

::

    'leftovers' matched 30 rows

.. code:: ipython

    found = data.str.extractall(DATE)
    total_found = len(found.date)

    print("Total Found: {}".format(total_found))
    print("Remaining: {}".format(len(data) - total_found))
    print("Discrepancy: {}".format(total_found - (numeric_count
                                                  + twentieth_count
                                                  + words_count
                                                  + backwards_count
                                                  + contraction
                                                  + no_day_count
                                                  + no_day_numeric_count
                                                  + year_only_count
                                                  + leftovers_count)))

::

    Total Found: 500
    Remaining: 0
    Discrepancy: 0

.. code:: ipython

    missing = [label for label in data.index if label not in found.index.levels[0]]
    try:
        print(missing[0], data.loc[missing[0]])
    except IndexError:
        print("all rows matched")

::

    all rows matched

6 Unifying the Formats
----------------------

To make it simpler, I'm going to use the ``mm/dd/yyyy`` format for the dates. I'm going to use the extracted series to avoid having different clean-up cases contaminating each other - e.g. dealing with 'January' when the day comes first as opposed to when the month comes first.

6.1 Helper Functions
~~~~~~~~~~~~~~~~~~~~

6.1.1 Clean
^^^^^^^^^^^

This is a generic function to clean up some data. I was initially using it directly, but for cases where the expression and replacement function are used more than once, there are helper functions to make it easier.

.. code:: ipython

    def clean(source, expression, replacement, sample=5):
        """applies the replacement to the source

        as a side-effect shows sample rows before and after

        Args:
         source (pandas.Series): source of the strings
         expression (str): regular expression to match what to replace
         replacement: function or expression to replace the matching expression
         sample (int): number of randomly chosen examples to show

        Returns:
         pandas.Series: the source with the replacement applied to it
        """
        print("Random Sample Before:")
        print(source.sample(sample))
        cleaned = source.str.replace(expression, replacement)
        print("\nRandom Sample After:")
        print(cleaned.sample(sample))
        print("\nCount of cleaned: {}".format(len(cleaned)))
        assert len(source) == len(cleaned)
        return cleaned

6.1.2 Clean Punctuation
^^^^^^^^^^^^^^^^^^^^^^^

.. code:: ipython

    def clean_punctuation(source, sample=5):
        """removes punctuation

        Args:
         source (pandas.Series): data to clean
         sample (int): size of sample to show

        Returns:
         pandas.Series: source with punctuation removed
        """
        print("Cleaning Punctuation")
        if any(source.str.contains(PUNCTUATION)):
            source = clean(source, PUNCTUATION, EMPTY_STRING)
        return source

6.1.3 Convert Long Month Names to Three-Letter Names
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: ipython

    LONG_TO_SHORT = dict(January="Jan",
                         February="Feb",
                         March="Mar",
                         April="Apr",
                         May="May",
                         June="Jun",
                         July="Jul",
                         August="Aug",
                         September="Sep",
                         October="Oct",
                         November="Nov",
                         December="Dec")

    # it turns out there are spelling errors in the data so this has to be fuzzy
    LONG_TO_SHORT_EXPRESSION = OR.join([GROUP.format(month)
                                        + CLASS.format(LOWER_CASE)
                                        + ZERO_OR_MORE
                                        for month in LONG_TO_SHORT.values()])

    def long_month_to_short(match):
        """convert long month to short
    
        Args:
         match (re.Match): object matching a long month

        Returns:
         str: shortened version of the month
        """
        return match.group(match.lastindex)

This next function is the one you would actually use to make the conversion.

.. code:: ipython

    def convert_long_months_to_short(source, sample=5):
        """convert long month names to short
    
        Args:
         source (pandas.Series): data with months
         sample (int): size of sample to show

        Returns:
         pandas.Series: data with short months
        """
        return clean(source,
                     LONG_TO_SHORT_EXPRESSION,
                     long_month_to_short)

6.1.4 Add January 1 to year-only dates
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code:: ipython

    def add_month_date(match):
        """adds 01/01 to years

        Args:
         match (re.Match): object that only matched a 4-digit year

        Returns:
         str: 01/01/YYYY
        """
        return "01/01/" + match.group()

And now the function to actually call.

.. code:: ipython

    def add_january_one(source):
        """adds /01/01/ to year-only dates

        Args:
         source (pandas.Series): data with the dates

        Returns:
         pandas.Series: years in source with /01/01/ added
        """
        return clean(source, YEAR_ONLY, add_month_date)

6.1.5 Two-Digit Numbers
^^^^^^^^^^^^^^^^^^^^^^^

This makes sure that there are exactly two digits in a number, adding a leading zero if needed.

.. code:: ipython

    two_digit_expression = GROUP.format(ONE_OR_TWO_DIGITS) + POSITIVE_LOOKAHEAD.format(SLASH)

    def two_digits(match):
        """add a leading zero if needed

        Args:
         match (re.Match): match with one or two digits

        Returns:
         str: the matched string with leading zero if needed
        """
        # for some reason the string-formatting raises an error if it's a string
        # so cast it to an int
        return "{:02}".format(int(match.group()))

This is the function to call for the case where the number is followed by a slash (e.g. ``2/``).

.. code:: ipython

    def clean_two_digits(source, sample=5):
        """makes sure source has two-digits
    
        Args:
         source (pandas.Series): data with digit followed by slash
         sample (int): number of samples to show

        Returns:
         pandas.Series: source with digits coerced to two digits
        """
        return clean(source, two_digit_expression, two_digits, sample)

This is like ``clean_two_digits`` but it doesn't check for the trailing slash. Use this if you have an isolated column of numbers that need to be two-digits.

.. code:: ipython

    def clean_two_digits_isolated(source, sample=5):
        """cleans two digits that are standalone

        Args:
         source (pandas.Series): source of the data
         sample (int): number of samples to show

        Returns:
         pandas.Series: converted data
        """
        return clean(source, ONE_OR_TWO_DIGITS, two_digits, sample)

6.1.6 Cleaning Up Months
^^^^^^^^^^^^^^^^^^^^^^^^

These clean up and convert written months (e.g. change ``Aug`` to ``08``).

.. code:: ipython

    digits = ("{:02}".format(month) for month in range(1, 13))
    MONTH_TO_DIGITS = dict(zip(MONTH_PREFIXES, digits))
    SHORT_MONTHS_EXPRESSION = OR.join((GROUP.format(month) for month in MONTH_TO_DIGITS))
    def month_to_digits(match):
        """converts short month to digits

        Args:
         match (re.Match): object with short-month

        Returns:
         str: month as two-digit number (e.g. Jan -> 01)
        """
        return MONTH_TO_DIGITS[match.group()]

.. code:: ipython

    def convert_short_month_to_digits(source, sample=5):
        """converts three-letter months to two-digits

        Args:
         source (pandas.Series): data with three-letter months
         sample (int): number of samples to show

        Returns:
         pandas.Series: source with short-months coverted to digits
        """
        return clean(source,
                     SHORT_MONTHS_EXPRESSION,
                     month_to_digits,
                     sample)

This function runs the previous three and is the main one that should be used. The others can be run individually for troubleshooting, though.

.. code:: ipython

    def clean_months(source, sample=5):
        """clean up months (which start as words)

        Args:
         source (pandas.Series): source of the months
         sample (int): number of random samples to show
        """
        cleaned = clean_punctuation(source)
    
        print("Converting long months to short")
        cleaned = clean(cleaned,
                        LONG_TO_SHORT_EXPRESSION,
                        long_month_to_short, sample)

        print("Converting short months to digits")
        cleaned = clean(cleaned,
                        SHORT_MONTHS_EXPRESSION,
                        month_to_digits, sample)
        return cleaned

6.1.7 Frame To Series
^^^^^^^^^^^^^^^^^^^^^

This is for the case where the date-fields were broken up into columns in a data-frame.

.. code:: ipython

    def frame_to_series(frame, index_source, samples=5):
        """re-combines data-frame into a series

        Args:
         frame (pandas.DataFrame): frame with month, day, year columns
         index_source (pandas.series): source to copy index from
         samples (index): number of random entries to print when done

        Returns:
         pandas.Series: series with dates as month/day/year
        """
        combined = frame.month + SLASH + frame.day + SLASH + frame.year
        combined.index = index_source.index
        print(combined.sample(samples))
        return combined

6.2 Year Only
~~~~~~~~~~~~~

For the case where there is only a year, I'll add January 1 to the dates.

.. code:: ipython

    year_only_cleaned = add_january_one(year_only)

::

    Random Sample Before:
         match
    472  0        2010
    495  0        1979
    497  0        2008
    481  0        1974
    486  0        1973
    Name: year_only, dtype: object

    Random Sample After:
         match
    495  0        01/01/1979
    470  0        01/01/1983
    462  0        01/01/1988
    481  0        01/01/1974
    480  0        01/01/2013
    Name: year_only, dtype: object

    Count of cleaned: 15

6.3 Leftovers
~~~~~~~~~~~~~

These were the odd cases that didn't seem to have a real pattern. Since I used a positive lookbehind to match everything but the year they only have the years in them, like the previous year-only cases.

.. code:: ipython

    leftovers_cleaned = add_january_one(leftovers)

::

    Random Sample Before:
         match
    487  0        1992
    477  0        1994
    498  0        2005
    488  0        1977
    484  0        2004
    Name: leftovers, dtype: object

    Random Sample After:
         match
    464  0        01/01/2016
    455  0        01/01/1984
    465  0        01/01/1976
    475  0        01/01/2015
    498  0        01/01/2005
    Name: leftovers, dtype: object

    Count of cleaned: 30

.. code:: ipython

    cleaned = pandas.concat([year_only_cleaned, leftovers_cleaned])
    print(len(cleaned))

::

    45

6.4 No Day Numeric
~~~~~~~~~~~~~~~~~~

This is for the case where the date is formatted with slashes and there are no day-values. To make the months uniform I'm going to make them all two-digits first.

.. code:: ipython

    no_day_numeric_cleaned = clean_two_digits(no_day_numeric)

::

    Random Sample Before:
         match
    450  0         1/1994
    374  0        11/2000
    403  0        10/1981
    454  0         7/1982
    358  0         1/1983
    Name: no_day_numeric, dtype: object

    Random Sample After:
         match
    426  0        11/1984
    415  0        02/1973
    360  0        12/2008
    367  0        09/2001
    362  0        08/2003
    Name: no_day_numeric, dtype: object

    Count of cleaned: 112

Now I'll add the day.

.. code:: ipython

    no_day_numeric_cleaned = clean(no_day_numeric_cleaned,
                                   SLASH,
                                   lambda m: "/01/")

::

    Random Sample Before:
         match
    368  0        08/1986
    409  0        10/1994
    443  0        09/2000
    404  0        10/1986
    395  0        02/1977
    Name: no_day_numeric, dtype: object

    Random Sample After:
         match
    349  0        05/01/1987
    392  0        05/01/2000
    448  0        05/01/2010
    394  0        10/01/2001
    424  0        04/01/1979
    Name: no_day_numeric, dtype: object

    Count of cleaned: 112

And add it to the total.

.. code:: ipython

    original = len(cleaned)
    cleaned = pandas.concat([cleaned, no_day_numeric_cleaned])
    assert len(cleaned) == no_day_numeric_count + original

.. code:: ipython

    print(len(cleaned))

::

    157

6.5 No Day
~~~~~~~~~~

This is for cases like *Mar 2011* where no day was given. We're going to assume that it's the first day of the month for each case.

.. code:: ipython

    no_day_cleaned = clean_months(no_day)

::

    Cleaning Punctuation
    Random Sample Before:
         match
    261  0           Oct 1986
    269  0          July 1992
    280  0          July 1985
    295  0         March 1983
    339  0        March, 2005
    Name: no_day, dtype: object

    Random Sample After:
         match
    228  0        September 1985
    304  0              Mar 2002
    253  0              Feb 2016
    276  0            April 1986
    272  0              Feb 1993
    Name: no_day, dtype: object

    Count of cleaned: 115
    Converting long months to short
    Random Sample Before:
         match
    315  0             Jun 1976
    242  0             Nov 2010
    237  0        February 1976
    330  0           April 1988
    311  0        February 1995
    Name: no_day, dtype: object

    Random Sample After:
         match
    306  0        May 2004
    254  0        Aug 1979
    269  0        Jul 1992
    337  0        Dec 2007
    241  0        May 2004
    Name: no_day, dtype: object

    Count of cleaned: 115
    Converting short months to digits
    Random Sample Before:
         match
    268  0        Dec 2009
    298  0        Jan 1993
    296  0        Aug 1979
    270  0        May 2006
    320  0        Nov 2012
    Name: no_day, dtype: object

    Random Sample After:
         match
    246  0        07 1981
    286  0        01 2013
    263  0        09 1981
    276  0        04 1986
    247  0        05 1983
    Name: no_day, dtype: object

    Count of cleaned: 115

Now we need to replace the spaces with the days.

.. code:: ipython

    no_day_cleaned = clean(no_day_cleaned,
                           SPACE + ONE_OR_MORE,
                           lambda match: "/01/")

::

    Random Sample Before:
         match
    251  0        12 1998
    290  0        12 2011
    281  0        08 2004
    308  0        02 1994
    294  0        02 1983
    Name: no_day, dtype: object

    Random Sample After:
         match
    304  0        03/01/2002
    332  0        06/01/1974
    310  0        10/01/1992
    293  0        09/01/2008
    322  0        10/01/1991
    Name: no_day, dtype: object

    Count of cleaned: 115

Now we can add it to the cleaned.

.. code:: ipython

    original = len(cleaned)
    cleaned = pandas.concat([cleaned, no_day_cleaned])
    print(len(cleaned))

::

    272

Now to make sure we're where we expect we are.

.. code:: ipython

    assert len(cleaned) == no_day_count + original

6.6 Contraction
~~~~~~~~~~~~~~~

There were no matches for the contraction so I'll ignore it for now. 

6.7 Backwards
~~~~~~~~~~~~~

This is the case where the day comes first. The first thing I'll do is split them up.

.. code:: ipython

    frame = pandas.DataFrame(backwards.str.split().tolist(),
                             columns="day month year".split())
    frame.head()

::

      day month  year
    0  24   Jan  2001
    1  10   Sep  2004
    2  26   May  1982
    3  28  June  2002
    4  06   May  1972

The next thing to do is to make sure the days all have two digits.

.. code:: ipython

    frame.day = clean_two_digits(frame.day)

::

    Random Sample Before:
    31    26
    39    21
    4     06
    57    13
    36    19
    Name: day, dtype: object

    Random Sample After:
    29    06
    68    18
    60    17
    11    11
    26    22
    Name: day, dtype: object

    Count of cleaned: 69

Next comes the months. This is basically the same problem as with the *no day* case so I'll re-use some of the code for that.


.. code:: ipython

    frame.month = clean_months(frame.month)

::

    Cleaning Punctuation
    Converting long months to short
    Random Sample Before:
    55    Dec
    41    Nov
    38    Jan
    54    Dec
    5     Oct
    Name: month, dtype: object

    Random Sample After:
    30    Oct
    55    Dec
    15    Feb
    38    Jan
    14    Oct
    Name: month, dtype: object

    Count of cleaned: 69
    Converting short months to digits
    Random Sample Before:
    29    Mar
    22    May
    45    Jan
    47    Aug
    61    Oct
    Name: month, dtype: object

    Random Sample After:
    16    05
    32    02
    4     05
    68    01
    38    01
    Name: month, dtype: object

    Count of cleaned: 69

Now we need to combine them back together. In hindsight it might have been easier to convert everything into data frames instead of the other way around. Or maybe not. Since we want the indexes from the original data as our final answer I also have to copy the index from the original series

.. code:: ipython

    backwards_cleaned = frame_to_series(frame, backwards)

::

         match
    177  0        01/18/1990
    128  0        06/28/2002
    181  0        08/18/1995
    158  0        08/23/2000
    185  0        08/17/1985
    dtype: object

No it gets added to the combined series.

.. code:: ipython

    original = len(cleaned)
    cleaned = pandas.concat([cleaned, backwards_cleaned])
    assert len(cleaned) == original + backwards_count

.. code:: ipython

    print(len(cleaned))

::

    341

6.8 Words
~~~~~~~~~

Since working with the data frame was easier than I though it would be I'll do that again.

.. code:: ipython

    frame = pandas.DataFrame(words.str.split().tolist(), columns="month day year".split())
    print(frame.head())

::

          month  day  year
    0     April  11,  1990
    1       May  30,  2001
    2       Feb  18,  1994
    3  February  18,  1981
    4  October.  11,  2013

First we'll clean out the months.

.. code:: ipython

    frame.month = clean_months(frame.month)

::

    Cleaning Punctuation
    Random Sample Before:
    25          Dec
    10         Mar.
    17        April
    14    September
    0         April
    Name: month, dtype: object

    Random Sample After:
    5         Jan
    12    October
    24        May
    2         Feb
    28        May
    Name: month, dtype: object

    Count of cleaned: 34
    Converting long months to short
    Random Sample Before:
    11       Jan
    13    August
    20       Sep
    6       July
    17     April
    Name: month, dtype: object

    Random Sample After:
    27    Oct
    30    Jul
    6     Jul
    14    Sep
    33    Sep
    Name: month, dtype: object

    Count of cleaned: 34
    Converting short months to digits
    Random Sample Before:
    24    May
    31    Jun
    5     Jan
    7     Dec
    32    Jan
    Name: month, dtype: object

    Random Sample After:
    15    07
    12    10
    1     05
    30    07
    21    08
    Name: month, dtype: object

    Count of cleaned: 34

Now we'll clean up the punctuation for the days.

.. code:: ipython

    frame.day = clean_punctuation(frame.day)

::

    Cleaning Punctuation
    Random Sample Before:
    22    11,
    13     12
    29     14
    16    11,
    24    14,
    Name: day, dtype: object

    Random Sample After:
    2     18
    1     30
    24    14
    15    25
    17    17
    Name: day, dtype: object

    Count of cleaned: 34

So, what do we have so far?

.. code:: ipython

    frame.head()

::

      month day  year
    0    04  11  1990
    1    05  30  2001
    2    02  18  1994
    3    02  18  1981
    4    10  11  2013

At this point we need to combine everything with a slash and restore the index.

.. code:: ipython

    words_cleaned = frame_to_series(frame, words)

::

         match
    194  0        04/11/1990
    217  0        06/13/2011
    209  0        07/25/1983
    216  0        11/11/1988
    223  0        10/14/1974
    dtype: object

Now we'll add it to the total.

.. code:: ipython

    original = len(cleaned)
    cleaned = pandas.concat([cleaned, words_cleaned])
    assert len(cleaned) == original + words_count
    print(len(cleaned))

::

    375

6.9 Twentieth Century
~~~~~~~~~~~~~~~~~~~~~

We'll do the same trick with creating a dataframe. The first thing, though, is to replace the dashes with slashes.

.. code:: ipython

    print(twentieth.iloc[21])
    twentieth_cleaned = twentieth.str.replace(DASH, SLASH)
    print(cleaned.iloc[21])

::

    4-13-82
    01/01/1991

Now, we'll create the frame.

.. code:: ipython

    frame = pandas.DataFrame(twentieth_cleaned.str.split(SLASH).tolist(),
                             columns=["month", "day", "year"])
    print(frame.head())

::

      month day year
    0    03  25   93
    1     6  18   85
    2     7   8   71
    3     9  27   75
    4     2   6   96

6.9.1 Months
^^^^^^^^^^^^

The months need to be converted to two-digits.

.. code:: ipython

    frame.month = clean_two_digits_isolated(frame.month)

::

    Random Sample Before:
    73     4
    53    10
    84     8
    93     6
    80    10
    Name: month, dtype: object

    Random Sample After:
    76    03
    33    07
    32    01
    94    07
    67    05
    Name: month, dtype: object

    Count of cleaned: 100

As do the days.

.. code:: ipython

    frame.day = clean_two_digits_isolated(frame.day)

::

    Random Sample Before:
    78    14
    29    15
    37    15
    75    18
    80    05
    Name: day, dtype: object

    Random Sample After:
    35    14
    30    14
    17    21
    88    16
    0     25
    Name: day, dtype: object

    Count of cleaned: 100

.. code:: ipython

    frame.head()

::

      month day year
    0    03  25   93
    1    06  18   85
    2    07  08   71
    3    09  27   75
    4    02  06   96

Now we have to add ``19`` to each of the years.

.. code:: ipython

    frame.year = clean(frame.year, TWO_DIGITS, lambda match: "19" + match.group())

::

    Random Sample Before:
    41    75
    90    97
    97    90
    69    97
    65    81
    Name: year, dtype: object

    Random Sample After:
    4     1996
    44    1971
    11    1975
    17    1998
    61    1979
    Name: year, dtype: object

    Count of cleaned: 100

Now we have to join them back up.

.. code:: ipython

    twentieth_cleaned = frame_to_series(frame, twentieth)

::

        match
    67  0        07/06/1991
    88  0        12/08/1982
    4   0        02/06/1996
    40  0        07/29/1975
    72  0        07/11/1977
    dtype: object

.. code:: ipython

    original = len(cleaned)
    cleaned = pandas.concat([cleaned, twentieth_cleaned])

.. code:: ipython

    assert len(cleaned) == original + twentieth_count

6.10 Numeric
~~~~~~~~~~~~

The final category is dates with the format ``mm/dd/yyyy``.

.. code:: ipython

    print(numeric.head())

::

        match
    14  0         5/24/1990
    15  0         1/25/2011
    17  0        10/13/1976
    24  0        07/25/1984
    30  0        03/31/1985
    Name: numeric, dtype: object

We should check and make sure there are no dashes here.

.. code:: ipython

    has_dashes = numeric.str.contains(DASH)
    print(numeric[has_dashes])

::

    Series([], Name: numeric, dtype: object)

It looks like it doesn't so we'll skip this check.

.. code:: ipython

    frame = pandas.DataFrame(numeric.str.split(SLASH).tolist(),
                             columns="month day year".split())
    print(frame.head())

::

      month day  year
    0     5  24  1990
    1     1  25  2011
    2    10  13  1976
    3    07  25  1984
    4    03  31  1985

.. code:: ipython

    frame.month = clean_two_digits_isolated(frame.month)

::

    Random Sample Before:
    5      5
    18    04
    4     03
    0      5
    10    12
    Name: month, dtype: object

    Random Sample After:
    0     05
    24    04
    3     07
    11    08
    13    11
    Name: month, dtype: object

    Count of cleaned: 25

.. code:: ipython

    frame.day = clean_two_digits_isolated(frame.day)

::

    Random Sample Before:
    9     11
    19    08
    8     15
    13     3
    24    27
    Name: day, dtype: object

    Random Sample After:
    23    20
    22    11
    7     13
    18    08
    0     24
    Name: day, dtype: object

    Count of cleaned: 25

.. code:: ipython

    numeric_cleaned = frame_to_series(frame, numeric)

::

        match
    94  0        12/08/1990
    92  0        04/08/2004
    43  0        04/13/2002
    38  0        07/27/1986
    14  0        05/24/1990
    dtype: object

.. code:: ipython

    original = len(cleaned)
    cleaned = pandas.concat([cleaned, numeric_cleaned])
    assert len(cleaned) == original + numeric_count
    print(len(cleaned))

::

    500

At this point it looks like we've cleaned all the cases.

6.11 Re-combining The Cleaned
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Because these notebooks can execute things out of order I'm going to create one monolithic concatenation and ignore the one that I was using to keep the running total.

.. code:: ipython

    cleaned = pandas.concat([numeric_cleaned,
                             twentieth_cleaned,
                             words_cleaned,
                             backwards_cleaned,
                             no_day_cleaned,
                             no_day_numeric_cleaned,
                             year_only_cleaned,
                             leftovers_cleaned,
    ])
    print(len(cleaned))
    print(cleaned.head())
    assert len(cleaned) == len(data)

::

    500
        match
    14  0        05/24/1990
    15  0        01/25/2011
    17  0        10/13/1976
    24  0        07/25/1984
    30  0        03/31/1985
    dtype: object

7 Convert to Datetimes
----------------------

.. code:: ipython

    print(cleaned.head())
    datetimes = pandas.to_datetime(cleaned, format="%m/%d/%Y")
    print(datetimes.head())

::

        match
    14  0        05/24/1990
    15  0        01/25/2011
    17  0        10/13/1976
    24  0        07/25/1984
    30  0        03/31/1985
    dtype: object
        match
    14  0       1990-05-24
    15  0       2011-01-25
    17  0       1976-10-13
    24  0       1984-07-25
    30  0       1985-03-31
    dtype: datetime64[ns]

.. code:: ipython

    sorted_dates = datetimes.sort_values()
    print(sorted_dates.head())

::

        match
    9   0       1971-04-10
    84  0       1971-05-18
    2   0       1971-07-08
    53  0       1971-07-11
    28  0       1971-09-12
    dtype: datetime64[ns]

.. code:: ipython

    print(sorted_dates.tail())

::

         match
    231  0       2016-05-01
    141  0       2016-05-30
    186  0       2016-10-13
    161  0       2016-10-19
    413  0       2016-11-01
    dtype: datetime64[ns]

The grader wants a Series with the indices of the original data put in the order of the sorted dates.

.. code:: ipython

    answer = pandas.Series(sorted_dates.index.labels[0])
    print(answer.head())

::

    0     9
    1    84
    2     2
    3    53
    4    28
    dtype: int16

8 The date\_sorter Function
---------------------------

This is the function called by the grader. Since the work was done outside of it we just need to make sure that it returns our answer.

.. code:: ipython

    def date_sorter():
        return answer

**note:** This produced a 94% score, so there are still some cases not correctly handled.
