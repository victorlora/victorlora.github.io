---
layout: post
title: Leap Years 
tags: 
    - Leap Years
    - smartGWT
    - DateItem
---

&ensp;&ensp; At work the other day, I came across an interesting situation which, at first glance, seemed really simple, but actually ended up being quite involved. I was working on a portal module and needed a way for the user to schedule a yearly task. My company uses <a href="http://www.smartclient.com/smartgwt/showcase/">smartGWT</a>, a library that offers a range of tools that greatly facilitate portal development and form creation, so the smartGWT <a href="http://www.java2s.com/Code/Java/GWT/UsingDateItemtocreatedatepickerSmartGWT.htm">DateItem</a> was the first logical choice for date selection.

&ensp;&ensp; The default DateItem consists of three selectors (month, day, year); however, due to the fact that this was going to be an yearly task, including the year as one of the options would have been rather pointless, so I decided I was only going to allow month-day selection. But, after some digging around the DateItem class, I discovered there was no way to remove the year. Thankfully, smartGWT easily allows you to create custom form items, so I decided I would create a form item composed of two selectors (month, day) – that's when things got interesting. Without the year, there was no way to determine whether the month of February should contain 28 or 29 days. Additionally, without the year, how would I store a date in the database? In order to answer these questions, I discussed the matter with one of my co-workers, and, after some discussion, we decided to look at how other companies handled the scheduling of yearly tasks on February 29th.

&ensp;&ensp; First, we looked at how our iPhones handled the situation. We scheduled something for February 29th on a leap year (2016) and set it to repeat yearly. What we found is that Apple actually handles this rather poorly. The task you scheduled to repeat yearly actually gets ignored for non-leap years and only shows up on the next leap year (4 years later). My thought would be that if you, for whatever reason, decide to schedule a task on February 29th and want it to repeat yearly, then Apple should either change the "Repeat yearly" to "Repeat every four years", or they should provide some sort of option to change the task to happen on either February 28th or March 1st on non-leap years. But alas, they don't, and I'm sure there was a long discussion over it when they decided to handle it this way. Then, out of curiosity, I decided to see how people born on February 29th celebrated their birthdays and found that for the most part they simply either celebrate their birthdays on the 28th or the 1st. So, armed with this information, we decided that if Apple was ok with simply ignoring tasks scheduled on February 29th on non-leap years and people were okay with celebrating their birthdays a day before or after, then we were okay simply disallowing February 29th from being an option, and in regards to the issue of storing the date in the database, we decided that we would store the day of the year instead.

The resulting custom date item was as follows:
{% highlight java %}
public class MonthDayPickerItem extends CanvasItem
{
	private DynamicForm form = new DynamicForm();
	private SelectItem monthPicker;
	private SelectItem dayPicker;
	private int[] daysInMonthArray 
            = {31, 28, 31, 30, 31, 30, 31, 31, 30 , 31, 30, 31};

	public MonthDayPickerItem(String name, String title)
	{
        buildForm();

        setTitle(title);
        setName(name); 
        setCanvas(form);
	}

	public void buildForm()
	{
        monthPicker = new SelectItem("month");
        monthPicker.setShowTitle(false);
        monthPicker.setWidth(50);
        monthPicker.setAllowEmptyValue(true);
        monthPicker.setChangedHandler(monthChangedHandler());
        monthPicker.setValueMap(buildMonthPickerValueMap());

        dayPicker = new SelectItem("day");
        dayPicker.setShowTitle(false);
        dayPicker.setWidth(50);
        dayPicker.setDisabled(true);

        form.setNumCols(2);
        form.setWidth(100);
        form.setItems(monthPicker, dayPicker);
	}

    private ChangedHandler monthChangedHandler()
    {
        ChangedHandler changedHandler = new ChangedHandler()
        {
            @Override
            public void onChanged(ChangedEvent event)
            {
                if (monthPicker.getValue() != null)
                {
                    if (dayPicker.getDisabled())
                        dayPicker.enable();

                    dayPicker.clearValue();
                    dayPicker.setValueMap(buildDayPickerValueMap());
                    dayPicker.setDefaultToFirstOption(true);
                }
                else if (monthPicker.getValue() == null 
                        && !dayPicker.getDisabled())
                {
                    dayPicker.clearValue();
                    dayPicker.disable();
                }
            }
        };

        return changedHandler;
    }

    private LinkedHashMap<Integer, String> buildMonthPickerValueMap()
    {
        LinkedHashMap<Integer, String> valueMap 
                = new LinkedHashMap<Integer, String>();
        valueMap.put(0, "Jan");
        valueMap.put(1, "Feb");
        valueMap.put(2, "Mar");
        valueMap.put(3, "Apr");
        valueMap.put(4, "May");
        valueMap.put(5, "Jun");
        valueMap.put(6, "Jul");
        valueMap.put(7, "Aug");
        valueMap.put(8, "Sep");
        valueMap.put(9, "Oct");
        valueMap.put(10, "Nov")
        valueMap.put(11, "Dec");

        return valueMap;
    }

    private LinkedHashMap<Integer, String> buildDayPickerValueMap()
    {
        int month = (int) monthPicker.getValue();

        LinkedHashMap<Integer, String> valueMap 
                = new LinkedHashMap<Integer, String>();
        for (int i = 1; i <= daysInMonthArray[month]; i++)
        {
            dayMap.put(i, String.valueOf(i));
        }

        return valueMap;
    }

    @Override
    public Object getValue()
    {
        if (monthPicker.getValue() != null 
                && dayPicker.getValue() != null)
        {
        	int dayOfYear = 0;

            int month = (int) monthPicker.getValue();
            for (int i = 0; i < month; i++)
            {
                dayOfYear += daysInMonthArray[i];
            }

            day += (int) dayPicker.getValue();

            return dayOfYear;
        }

        return null
    }

    public void setValues(Record monthDay)
    {
        if (monthDay != null)
        {
            Integer month = monthDay.getAttributeAsInt("month");
            Integer day = monthDay.getAttributeAsInt("day");

            if (month != null && day != null)
            {
                monthPicker.setValue(month);

                dayPicker.setValue(day);
                dayPicker.enable();
                dayPicker.setValueMap(buildDayPickerValueMap());
            }
        }
    }
}
{% endhighlight %}

&ensp;&ensp; As you may have noted, this class converts the users selection of month and day into a day of year in the getValue() method. Now the last piece of the puzzle is converting that day of year back into a month and day so that the MonthDayPicker reflects the database value correctly.

&ensp;&ensp; Below is the class that takes a day of the year and converts back to a month and day. Note that, in order to preserve the users original month day choice, the converter offsets the extra day in leap years by adding one to any day of year greater than or equal to February 29th (i.e. On non-leap years, the user's choice of March 1st is equal to the 60th day of the year; however, on a leap year the 60th day of the year is February 29th. The converter takes this into account by adding 1 to any day of year >= 60. So, on a leap year a user's choice of March 1st would be stored in the database as day of year 60, but would be returned as day of year 61).
{% highlight java %}
public class DayOfYearToMonthDayConverter
{
    public static MonthDay convert(Integer dayOfYear)
    {
        MonthDay monthDay = new MonthDay();
        if (dayOfYear != null)
        {
            Calendar calendar = Calendar.getInstance();

            int year = calendar.get(Calendar.YEAR);
            boolean isLeapYear = ((year % 4 == 0) && (year % 100 != 0) 
                    || (year % 400 == 0));

            // if current year is a leap year and dayOfYear comes after Feb. 29th
            // add 1 to dayOfYear to offset extra day
            if (isLeapYear && dayOfYear >= 60)
            {
                dayOfYear++;
            }

            calendar.setTime(new Date());
            calendar.set(year, 0, 0);
            calendar.add(Calendar.DAY_OF_YEAR, dayOfYear);

            int month = calendar.get(Calendar.MONTH);
            int day = calendar.get(Calendar.DAY_OF_MONTH);

            monthDay.setMonth(month);
            monthDay.setDay(day);
        }

        return monthDay;
    }
}
{% endhighlight %}

{% highlight java %}
public class MonthDay
{
    private Integer month;
    private Integer day;

    public Integer getMonth()
    {
        return this.month;
    }

    public void setMonth(Integer month)
    {
        this.month = month;
    }

    public Integer getDay()
    {
        return this.day;
    }

    public void setDay(Integer day)
    {
        this.day = day;
    }
}
{% endhighlight %}

&ensp;&ensp; So, that's how I handled leap years without the need of having the user specify a year. I'm sure there were plenty of other ways of going about the matter and perhaps one day the need to move away from this method will arise, but for now it serves our purposes well. Thank you for reading this post and I hope that, if you've stumbled upon this while searching for ways to handle leap years in smartGWT, you find this somewhat helpful.
