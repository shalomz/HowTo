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
