# HowTo
A Curated list on How-To's, ranging from any and all JS, Python, Django.
# 1. How to Render Datepicker Widget on Multiple Fields
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
