# Handling recurring events in a Django calendar app

The key part of your problem is that you're using a handful of models to track
just one concept, so you're introducing a lot of duplication and complexity.
Each of the additional models is a "type" of `Lesson`, so you should be using
inheritance here. Additionally, most of the additional models are merely
tracking a particular characteristic of a `Lesson`, and as a result should not
actually be models themselves. This is how I would have set it up:

    
    
    class Lesson(models.Model): RECURRENCE_CHOICES = ( (0, 'None'), (1, 'Daily'), (7, 'Weekly'), (14, 'Biweekly') ) student = models.ForeignKey(Student) frequency = models.IntegerField(choices=RECURRENCE_CHOICES) lessonTime = models.TimeField('Lesson Time') startDate = models.DateField('Start Date') endDate = models.DateField('End Date') cancelledDate = models.DateField('Cancelled Date', blank=True, null=True) paidAmt = models.DecimalField('Amount Paid', max_digits=5, decimal_places=2, blank=True, null=True) paidDate = models.DateField('Date Paid', blank=True, null=True) class CancelledLessonManager(models.Manager): def get_query_set(self): return self.filter(cancelledDate__isnull=False) class CancelledLesson(Lesson): class Meta: proxy = True objects = CancelledLessonManager() class PaidLessonManager(models.Manager): def get_query_set(self): return self.filter(paidDate__isnull=False) class PaidLesson(Lesson): class Meta: proxy = True objects = PaidLessonManager()
    

You'll notice that I moved all the attributes onto `Lesson`. This is the way
it should be. For example, `Lesson` has a `cancelledDate` field. If that field
is NULL then it's not cancelled. If it's an actual date, then it is cancelled.
There's no need for another model.

However, I have left both `CancelledLesson` and `PaidLesson` for instructive
purposes. These are now what's called in Django "proxy models". They don't get
their own database table (so no nasty data duplication). They're purely for
convenience. Each has a custom manager to return the appropriate matching
`Lessons`, so you can do `CancelledLesson.objects.all()` and get only those
`Lesson`s that are cancelled, for example. You can also use proxy models to
create unique views in the admin. If you wanted to have an administration area
only for `CancelledLesson`s you can, while all the data still goes into the
one table for `Lesson`.

`CompositeLesson` is gone, and good riddance. This was a product of trying to
compose these three other models into one cohesive thing. That's no longer
necessary, and your queries will be _dramatically_ easier as a result.

**EDIT**

I neglected to mention that you can and should add utility methods to the
`Lesson` model. For example, while tracking cancelled/not by whether the field
is NULL or not makes sense from a database perspective, from programming
perspective it's not as intuitive as it could be. As a result, you might want
to do things like:

    
    
    @property
    def is_cancelled(self): return self.cancelledDate is not None ... if lesson.is_cancelled: print 'This lesson is cancelled'
    

Or:

    
    
    import datetime ... def cancel(self, date=None, commit=True): self.cancelledDate = date or datetime.date.today() if commit: self.save()
    

Then, you can cancel a lesson simply by calling `lesson.cancel()`, and it will
default to cancelling it today. If you want to future cancel it, you can pass
a date: `lesson.cancel(date=tommorrow)` (where `tomorrow` is a `datetime`). If
you want to do other processing before saving, pass `commit=False`, and it
won't actually save the object to the database yet. Then, call `lesson.save()`
when you're ready.
