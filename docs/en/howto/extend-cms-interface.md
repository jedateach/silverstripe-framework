# How to extend the CMS interface #

## Introduction ##

The CMS interface works just like any other part of your website: It consists of PHP controllers,
templates, CSS stylesheets and JavaScript. Because it uses the same base elements,
it is relatively easy to extend. 
As an example, we're going to add a permanent "bookmarks" bar to popular pages at the bottom of the CMS.
A page can be bookmarked by a CMS author through a simple checkbox.

For a deeper introduction to the inner workings of the CMS, please refer to our
guide on [CMS Architecture](../reference/cms-architecture).

## Overload a CMS template ##

If you place a template with an identical name into your application template directory
(usually `mysite/templates/`), it'll take priority over the built-in one.

CMS templates are inherited based on their controllers, similar to subclasses of
the common `Page` object (a new PHP class `MyPage` will look for a `MyPage.ss` template).
We can use this to create a different base template with `LeftAndMain.ss`
(which corresponds to the `LeftAndMain` PHP controller class).

Copy the template markup of the base implementation at `framework/admin/templates/LeftAndMain.ss` into
`mysite/templates/LeftAndMain.ss`. It will automatically be picked up by the CMS logic. Add a new section after the
`$Content` tag:
	
	:::ss
	...
	<div class="cms-container" data-layout-type="border">
		$Menu
		$Content
		<div class="cms-bottom-bar south">
			<ul>
				<li><a href="admin/page/edit/show/1">Edit "My popular page"</a></li>
				<li><a href="admin/page/edit/show/99">Edit "My other page"</a></li>
			</ul>
		</div>
	</div>
	...
	
Refresh the CMS interface with `admin/?flush=all`, and you should see the new bottom bar with some hardcoded links.
We'll make these dynamic further down. 

You might have noticed that we didn't write any JavaScript to add our layout manager. 
The important piece of information is the `south` class in our new `<div>` structure,
plus the height value in our CSS. It instructs the existing parent layout how to render the element.
This layout manager ([jLayout](http://www.bramstein.com/projects/jlayout/)) 
allows us to build complex layouts with minimal JavaScript configuration.

See [layout reference](../reference/layout) for more specific information on CMS layouting.
	
## Include custom CSS in the CMS

In order to show the links in one line, we'll add some CSS, and get it to load with the CMS interface.
Paste the following content into a new file called `mysite/css/BookmarkedPages.css`:

	:::css
	.cms-bottom-bar {height: 20px; padding: 5px; background: #C6D7DF;}
	.cms-bottom-bar ul {list-style: none; margin: 0; padding: 0;}
	.cms-bottom-bar ul li {float: left; margin-left: 1em;}
	.cms-bottom-bar a {color: #444444;}

Load the new CSS file into the CMS, by setting the `LeftAndMain.extra_requirements_css`
[configuration value](/topics/configuration) to 'mysite/css/BookmarkedPages.css'.

## Create a "bookmark" flag on pages ##

Now we'll define which pages are actually bookmarked, a flag that is stored in the database.
For this we need to decorate the page record with a `DataExtension`.
Create a new file called `mysite/code/BookmarkedPageExtension.php` and insert the following code.

	:::php
	<?php
	class BookmarkedPageExtension extends DataExtension {
		private static $db = array('IsBookmarked' => 'Boolean');
		
		public function updateCMSFields(FieldList $fields) {
			$fields->addFieldToTab('Root.Main',
				new CheckboxField('IsBookmarked', "Show in CMS bookmarks?")
			);
		}
	}

Enable the extension in your [configuration file](/topics/configuration)

	:::yml
	SiteTree:
	  extensions:
	    - BookmarkedPageExtension

In order to add the field to the database, run a `dev/build/?flush=all`.
Refresh the CMS, open a page for editing and you should see the new checkbox.

## Retrieve the list of bookmarks from the database

One piece in the puzzle is still missing: How do we get the list of bookmarked
pages from the datbase into the template we've already created (with hardcoded links)? 
Again, we extend a core class: The main CMS controller called `LeftAndMain`.

Add the following code to a new file `mysite/code/BookmarkedLeftAndMainExtension.php`;

	:::php
	<?php
	class BookmarkedPagesLeftAndMainExtension extends LeftAndMainExtension {
		public function BookmarkedPages() {
			return Page::get()->filter("IsBookmarked", 1);
		}
	}
	
Enable the extension in your [configuration file](/topics/configuration)

	:::yml
	LeftAndMain:
	  extensions:
	    - BookmarkedPagesLeftAndMainExtension

As the last step, replace the hardcoded links with our list from the database.
Find the `<ul>` you created earlier in `mysite/admin/templates/LeftAndMain.ss`
and replace it with the following:

	:::ss
	<ul>
		<% loop BookmarkedPages %>
		<li><a href="admin/pages/edit/show/$ID">Edit "$Title"</a></li>
		<% end_loop %>
	</ul>

## Extending the CMS actions

CMS actions follow a principle similar to the CMS fields: they are built in the backend with the help of `FormFields`
and `FormActions`, and the frontend is responsible for applying a consistent styling.

The following conventions apply:

* New actions can be added by redefining `getCMSActions`, or adding an extension with `updateCMSActions`.
* It is required the actions are contained in a `FieldSet` (`getCMSActions` returns this already).
* Standalone buttons are created by adding a top-level `FormAction` (no such button is added by default).
* Button groups are created by adding a top-level `CompositeField` with `FormActions` in it.
* A `MajorActions` button group is already provided as a default.
* Drop ups with additional actions that appear as links are created via a `TabSet` and `Tabs` with `FormActions` inside.
* A `ActionMenus.MoreOptions` tab is already provided as a default and contains some minor actions.
* You can override the actions completely by providing your own `getAllCMSFields`.

Let's walk through a couple of examples of adding new CMS actions in `getCMSActions`.

First of all we can add a regular standalone button anywhere in the set. Here we are inserting it in the front of all
other actions. We could also add a button group (`CompositeField`) in a similar fashion.

	:::php
	$fields->unshift(FormAction::create('normal', 'Normal button'));

We can affect the existing button group by manipulating the `CompositeField` already present in the `FieldList`.

	:::php
	$fields->fieldByName('MajorActions')->push(FormAction::create('grouped', 'New group button'));

Another option is adding actions into the drop-up - best place for placing infrequently used minor actions.

	:::php
	$fields->addFieldToTab('ActionMenus.MoreOptions', FormAction::create('minor', 'Minor action'));

We can also easily create new drop-up menus by defining new tabs within the `TabSet`.

	:::php
	$fields->addFieldToTab('ActionMenus.MyDropUp', FormAction::create('minor', 'Minor action in a new drop-up'));

<div class="hint" markdown='1'>
Empty tabs will be automatically removed from the `FieldList` to prevent clutter.
</div>

New actions will need associated controller handlers to work. You can use a `LeftAndMainExtension` to provide one. Refer
to [Controller documentation](../topics/controller) for instructions on setting up handlers.

To make the actions more user-friendly you can also use alternating buttons as detailed in the [CMS Alternating
Button](../reference/cms-alternating-button) how-to.

## Summary

In a few lines of code, we've customized the look and feel of the CMS.
While this example is only scratching the surface, it includes most building
blocks and concepts for more complex extensions as well.

## Related

 * [Reference: CMS Architecture](../reference/cms-architecture)
 * [Reference: Layout](../reference/layout)
 * [Topics: Rich Text Editing](../topics/rich-text-editing)
 * [CMS Alternating Button](../reference/cms-alternating-button)
