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

## quick notes

- evts with (htmx.js | alpine.js). example: close-details event (no server needed)
- evts with HTTP response header. example: DB record deleted/updated...

## 2 pairs of trigger and hx-get

Not supported by htmx
Workaround:


````html
<button
  hx-on:click="htmx.ajax('GET','/loadA', {target:'#a'})"
  hx-on:mouseenter="htmx.ajax('GET','/loadB', {target:'#b'})">
  hx-trigger="my-event from: body"
  hx-get="..."  
  Load Stuff
</button>

````


Alternative:
NOTE: `hx-on:` implicitly listens on from:body

````html
<button
   hx-on:close-details-requested="htmx.ajax('POST', '/row/close-details', {
   source: this
  })"
   or:
   hx-on:close-details-requested="
    htmx.ajax('POST','/row/close-details', {
      values: { id: this.dataset.id }
    })
  "
</button>
````


````html
<tr
  hx-trigger="click"
  hx-target="next tr"
  hx-swap="delete">
  Click me to remove next row
</tr>

<tr>
  <td>Row to be deleted</td>
</tr>
````

### Using server roundtrip

````JS
export const EvtPersondetailsRow = (props: { vm: EvtPersonDetailModel, children: ComponentChildren }) => (
<tr
    hx-trigger="click"
    hx-swap="none"
    hx-get={props.vm._backLink}
>
    <td style="border-style: none">
        {/* Ugly: workaround for multiple trigger/action pairs:*/}
        {/* Since HTMX does not support multiple trigger/action pairs*/}
        {/* this <td> serves as a workaround on which further trigger/actions can be placed on.*/}
        {/* In addition this component should be unaware what happens when the click of <tr> happens*/}
        {/* but it should support the replacement of itself with something else (the standard row for this app)*/}
        {props.children}
    </td>
...
````

````JS
export const EvtPersonDetails = (props: { vm: OOBPersonDetailModel }) => (
<>
    <EvtPersondetailsRow vm={props.vm}>
        {/* Inject behavior for 'close-details-requested' event from here into EvtPersondetailsRow component.*/}
        {/* Use <template> since it is not rendered by the browser and therefore does not have undesired impact on layout.*/}
        <template
                hx-trigger={`close-details-requested[event.detail.id === ${props.vm.id}] from:body`}
                hx-get="/demo/event/person/{id}/row"
                hx-vals='js:{id: event.detail.id}'
                hx-target='closest tr'
                hx-swap="outerHTML"
        ></template>
    </EvtPersondetailsRow>
    <EvtPersondetailsCard vm={props.vm}/>
</>
);
...
````

````Java
@GetMapping(RouteBuilder.DETAILS_BACK_URL)
public void detailsBack(@PathVariable int id, HttpServletResponse response) {
	response.setHeader(HTMXConsts.HX_TRIGGER, """
		{"%s": {"id": %d}}\
		""".formatted(EvtConstants.CLOSE_DETAILS_REQUESTED, id));
}
````


## TODO

- htmx client-side only patterns with htmx,js,alpine.js,hyperscript


