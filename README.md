# HowTo
A Curated list on How-To's, ranging from any and all JS, Python, Django.
Feel free to use all the snippets here for whatever functionality/scenario. I hope this becomes helpful!
Contribute, contribute, contribute!
# 1. How to Render A Datepicker Widget on Multiple Input Fields
Say you want several input fields to have a datepicker widget, without explicitly defining the inputs for each one, or for the case where the input fields are generated dynamically, a possible fix (using jQuery) is written below:
::
    
    <script>
		$(document).ready(function () {
			$('body').on('focus',".dateinput", function(){
				$(this).datepicker();
			});
		})
	</script>
Whereby `dateinput` is the class of the input fields.
The complete markup is coming.
Where I found this useful: I had input fields that needed datepicker widgets within Django Formsets.

# 2. How to find if today is in a range of given dates
Say you hava an object with 2 Date attributes `start_date` and `end_date` and you would like to find the dates between the 2 dates:
:: 

	# Somewhere in your views.py
	from datetime import date, timedelta
	
	def daterange(start_date, end_date):
            """
            Function to find all dates within a particular range of 2 dates
   	    """
    	    for n in range(int ((end_date - start_date).days)+1):
            yield start_date + timedelta(n)
			
			
	def your_view(request):
	    # Logic here, for capturing the 2 dates and assigning them names, i.e. start_date and end_date
	     now = date.today()  #Finding today's date
	     if now in daterange(start_date, end_date): 
                # Logic for if the date today is among the dates
             else:
                # Logic for if date today is not among the dates
		

		
