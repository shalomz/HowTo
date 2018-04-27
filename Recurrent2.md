# Simple Django Event Calendar

[ 2011-06-13 ](https://williambert.online/2011/06/django-event-calendar-for-a
-django-beginner/)

I've been teaching myself Django by writing a web app that tracks reading
series in cities: [Readsr](http://www.readsrs.com/). A reading series is a
kind of recurring event. It defines a time, location, and recurrence rule such
as first Monday of the month. Writing a list view to display upcoming readings
was easy, but I also wanted to create a calendar view, similar to what Google
Calendar provides. That took a little more work.

### [](https://williambert.online/2011/06/django-event-calendar-for-a-django-
beginner/#Existing-Solutions)Existing Solutions

First I searched for existing Django calendar solutions. I found several.
[Swingtime](https://code.google.com/p/django-swingtime/) and [django-
agenda](https://github.com/dokterbob/django-agenda) seemed very well thought
out and comprehensive, but were also perhaps overkill for what I needed.
[django-gencal](https://code.google.com/p/django-gencal/) doesn't appear to be
maintained and I had trouble understanding the documentation (though you may
have better results, as I am slow).

### [](https://williambert.online/2011/06/django-event-calendar-for-a-django-
beginner/#A-Way-Forward)A Way Forward

I found that Python's calendar module has a built template called [HTMLCalenda
r](http://docs.python.org/library/calendar.html#calendar.HTMLCalendar), which
sounded promising. Then I found a
[couple](http://journal.uggedal.com/creating-a-flexible-monthly-calendar-in-
django/) [examples](http://drumcoder.co.uk/blog/2010/jun/13/monthly-calendar-
django/) of people inheriting from HTMLCalendar to add data to the calendar
display. This sounded right on, so I adapted this code for my reading events.

### [](https://williambert.online/2011/06/django-event-calendar-for-a-django-
beginner/#Problem-Presentation-and-Content-Mixed)Problem: Presentation and
Content Mixed

I noticed a problem in the code I was adapting. The view was producing HTML.
That seemed to violate separation of content and presentation. Shouldn't the
HTML be generated in the template? And since the templating language itself
isn't powerful enough to generate an HTML table from a list of objects, that
meant I needed to write my own template tag. Yikes.

### [](https://williambert.online/2011/06/django-event-calendar-for-a-django-
beginner/#Writing-My-First-Template-Tag)Writing My First Template Tag

Django's documentation made it easy, though. A template tag consists of a few
parts. Below is the code; it goes into a subdirectory of the django app called
"templatetags."

The first part follows directly from the Django docs: get a register object.

Then I define a function to parse the template tag arguments and return the
node (the HTML code from which the page is eventually build). The template
syntax is defined here.

Then I define the node itself, which is made thread-safe by storing and
retrieving the variables passed through the template tag from a context
(again, this is straight from the Django docs).

Then I inherit from HTMLCalendar and redefine the format methods to add the
particular reading event data. You could adapt this class to any kind of event
that has an associated date/time by changing the groupby lambda function to
use whatever field your event object uses to store its date and time (my
reading object simply calls it "date_and_time").

Finally, I register this template tag so it is available to templates.

Here's the code.

Reading calendar

    
    
    1  
    2  
    3  
    4  
    5  
    6  
    7  
    8  
    9  
    10  
    11  
    12  
    13  
    14  
    15  
    16  
    17  
    18  
    19  
    20  
    21  
    22  
    23  
    24  
    25  
    26  
    27  
    28  
    29  
    30  
    31  
    32  
    33  
    34  
    35  
    36  
    37  
    38  
    39  
    40  
    41  
    42  
    43  
    44  
    45  
    46  
    47  
    48  
    49  
    50  
    51  
    52  
    53  
    54  
    55  
    56  
    57  
    58  
    59  
    60  
    61  
    62  
    63  
    64  
    65  
    66  
    67  
    68  
    69  
    70  
    71  
    72  
    73  
    74  
    75  
    76  
    77  
    78  
    79  
    80  
    81  
    82  
    83  
    84  
    85  
    86  
    87  
    88  
    89  
    90  
    91  
    92  
    93  
    

|

    
    
       
    from calendar import HTMLCalendar  
    from django import template  
    from datetime import date  
    from itertools import groupby  
      
    from django.utils.html import conditional_escape as esc  
      
    register = template.Library()  
      
    def do_reading_calendar(parser, token):  
    	"""  
    	The template tag's syntax is { % reading_calendar year month reading_list %}  
    	"""  
      
    	try:  
     tag_name, year, month, reading_list = token.split_contents()  
    	except ValueError:  
     raise template.TemplateSyntaxError, "%r tag requires three arguments" % token.contents.split()[0]  
    	return ReadingCalendarNode(year, month, reading_list)  
    	  
      
    class ReadingCalendarNode(template.Node):  
    	"""  
    	Process a particular node in the template. Fail silently.  
    	"""  
    	  
    	def __init__(self, year, month, reading_list):  
     try:  
     self.year = template.Variable(year)  
     self.month = template.Variable(month)  
     self.reading_list = template.Variable(reading_list)  
     except ValueError:  
     raise template.TemplateSyntaxError  
       
    	def render(self, context):  
     try:  
       
     my_reading_list = self.reading_list.resolve(context)  
     my_year = self.year.resolve(context)  
     my_month = self.month.resolve(context)  
     cal = ReadingCalendar(my_reading_list)  
     return cal.formatmonth(int(my_year), int(my_month))  
     except ValueError:  
     return   
     except template.VariableDoesNotExist:  
     return  
      
      
    class ReadingCalendar(HTMLCalendar):  
    	"""  
    	Overload Python's calendar.HTMLCalendar to add the appropriate reading events to  
    	each day's table cell.  
    	"""  
    	  
    	def __init__(self, readings):  
     super(ReadingCalendar, self).__init__()  
     self.readings = self.group_by_day(readings)  
      
    	def formatday(self, day, weekday):  
     if day != 0:  
     cssclass = self.cssclasses[weekday]  
     if date.today() == date(self.year, self.month, day):  
     cssclass += ' today'  
     if day in self.readings:  
     cssclass += ' filled'  
     body = ['<ul>']  
     for reading in self.readings[day]:  
     body.append('<li>')  
     body.append('<a href="%s">' % reading.get_absolute_url())  
     body.append(esc(reading.series.primary_name))  
     body.append('</a></li>')  
     body.append('</ul>')  
     return self.day_cell(cssclass, '<span class="dayNumber">%d</span> %s' % (day, ''.join(body)))  
     return self.day_cell(cssclass, '<span class="dayNumberNoReadings">%d</span>' % (day))  
     return self.day_cell('noday', '&nbsp;')  
      
    	def formatmonth(self, year, month):  
     self.year, self.month = year, month  
     return super(ReadingCalendar, self).formatmonth(year, month)  
      
    	def group_by_day(self, readings):  
     field = lambda reading: reading.date_and_time.day  
     return dict(  
     [(day, list(items)) for day, items in groupby(readings, field)]  
     )  
      
    	def day_cell(self, cssclass, body):  
     return '<td class="%s">%s</td>' % (cssclass, body)  
      
      
    register.tag("reading_calendar", do_reading_calendar)  
      
      
  
---|---  
  
Then, here's the view (and a couple helper functions) that gets called with
the arguments from the URL, including the year, month, and series to display
events for.

View

    
    
    1  
    2  
    3  
    4  
    5  
    6  
    7  
    8  
    9  
    10  
    11  
    12  
    13  
    14  
    15  
    16  
    17  
    18  
    19  
    20  
    21  
    22  
    23  
    24  
    25  
    26  
    27  
    28  
    29  
    30  
    31  
    32  
    33  
    34  
    35  
    36  
    37  
    38  
    39  
    40  
    41  
    42  
    43  
    44  
    45  
    46  
    47  
    48  
    49  
    50  
    51  
    52  
    53  
    54  
    55  
    56  
    57  
    58  
    

|

    
    
    def named_month(month_number):  
      
    	"""  
    	Return the name of the month, given the number.  
    	"""  
    	return date(1900, month_number, 1).strftime("%B")  
    	  
    def this_month(request):  
    	"""  
    	Show calendar of readings this month.  
    	"""  
    	today = datetime.now()  
    	return calendar(request, today.year, today.month)  
    	  
    	  
    def calendar(request, year, month, series_id=None):  
    	"""  
    	Show calendar of readings for a given month of a given year.  
    	``series_id``  
    	The reading series to show. None shows all reading series.  
    	"""  
    	  
    	my_year = int(year)  
    	my_month = int(month)  
    	my_calendar_from_month = datetime(my_year, my_month, 1)  
    	my_calendar_to_month = datetime(my_year, my_month, monthrange(my_year, my_month)[1])  
      
    	my_reading_events = Reading.objects.filter(date_and_time__gte=my_calendar_from_month).filter(date_and_time__lte=my_calendar_to_month)  
    	if series_id:  
     my_reading_events = my_reading_events.filter(series=series_id)  
      
    	  
    	my_previous_year = my_year  
    	my_previous_month = my_month - 1  
    	if my_previous_month == 0:  
     my_previous_year = my_year - 1  
     my_previous_month = 12  
    	my_next_year = my_year  
    	my_next_month = my_month + 1  
    	if my_next_month == 13:  
     my_next_year = my_year + 1  
     my_next_month = 1  
    	my_year_after_this = my_year + 1  
    	my_year_before_this = my_year - 1  
    	return render_to_response("cal_template.html", { 'readings_list': my_reading_events,  
     'month': my_month,  
     'month_name': named_month(my_month),  
     'year': my_year,  
     'previous_month': my_previous_month,  
     'previous_month_name': named_month(my_previous_month),  
     'previous_year': my_previous_year,  
     'next_month': my_next_month,  
     'next_month_name': named_month(my_next_month),  
     'next_year': my_next_year,  
     'year_before_this': my_year_before_this,  
     'year_after_this': my_year_after_this,  
    	}, context_instance=RequestContext(request))  
      
      
  
---|---  
  
And finally, here's the template where we load the template tag and employ it,
passing the year, month, and list from the view (you would also want to write
some control elements that use previous_year, previous_month, etc. to allow
the user to change what the calendar displays, but because I want to wrap this
up I'll forgo writing that out):

Use template tag

    
    
    1  
    2  
    3  
    4  
    5  
    6  
    7  
    8  
    

|

    
    
      
    {% load reading_tags %}  
      
    <div id="calendar">  
    	{% reading_calendar year month reading_list %}  
    </div>  
      
      
      
  
---|---  
  
Hopefully that makes sense. Enjoy!

_You can also see this code (made generic for any kind of event) [on django
snippets](http://djangosnippets.org/snippets/2464/)._

Teilen [Kommentare](http://williambert.online/2011/06/django-event-calendar-
for-a-django-beginner/#disqus_thread)

[ **Neuer**

How to: Unit testing in Django with mocking and patching

](https://williambert.online/2011/07/how-to-unit-testing-in-django-with-
mocking-and-patching/) [ **Älter**

Marshall, Barnard, Laurel, Marshall

](https://williambert.online/2011/05/marshall-barnard-laurel-marshall/)
