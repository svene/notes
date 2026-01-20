# HTMX Tips and Tricks

## Override hx-swap on the server side
response.setHeader("HX-Reswap", "none");


## HTMX-Delete-Form to page redirect 
````html
<form id="person-form" hx-delete="/person/delete">
...
</form>
````

````java
@DeleteMapping("/person/delete")
public ResponseEntity<String> deleteRows3(@RequestParam List<Integer> selection, HttpServletResponse response) {
    peopleService.deleteByIds(selection);
    response.setHeader("HX-Redirect", PEOPLE_URL);
    return people();
}
````
TODO: combine with progressive enhancement variant (see HTMX book)

maybe like this:
````java
@DeleteMapping("/person/delete")
public RedirectView deleteRows2(@RequestParam List<Integer> selection, HttpServletRequest request, HttpServletResponse response) {
    peopleService.deleteByIds(selection);
    // make the browser redirect with a GET instead of a PUT/POST/DELETE:
    request.setAttribute(View.RESPONSE_STATUS_ATTRIBUTE, HttpStatus.SEE_OTHER); // 303 (See Other) instead of 302 (Found)
    return new RedirectView(PEOPLE_URL);
}
````

also see this in mybookmarks app (TODO: verify that this is only for fragment exchange):
````java
/**
 * NOTE:
 * no Post/Redirect/Get Pattern is needed with htmx
 * see https://htmx.org/docs/#response-headers:
 * "Submitting a form via htmx has the benefit of
 *  no longer needing the Post/Redirect/Get Pattern.
 *  After successfully processing a POST request on the server,
 *  you don’t need to return a HTTP 302 (Redirect).
 *  You can directly return the new HTML fragment."
 */
@PostMapping(path = URL)
public ViewContext doit(HttpServletResponse response) {
	var bm = bookmarkSessionService.store().getPreviewBookmark();
	if (!BookmarkUtil.isBookmarkValid(bm)) {
		HtmxResponseUtils.setHxReSwap(response, HxSwapValues.NONE);
		return new Nothing.Ctx();
	}
	addBookmark();
	return new Ctx(bookmarkSessionService);
}
````
TODO: in an example/pattern app show both: form with POST/DELETE for whole page replacement and only for fragment replacement


## Table with details and selection

open details on row (when user clicks on any child of the row):
````html
<tr
    style="cursor: pointer"
    hx-trigger="click"
    hx-target="this"
    hx-swap="outerHTML"
    hx-get={`/person/${props.vm.id}/details`}
>
...
</tr>
````
on checkbox <td> prevent click event to bubble up to row:
````html
<td onClick="{event.stopPropagation();}"><input type="checkbox" name="selection" value={props.vm.id}></input></td>
````

or:
````html
<td hx-trigger="click consume"><input type="checkbox" name="selection" value={props.vm.id}></input></td>
````


