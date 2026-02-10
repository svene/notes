# HONO-JSX Tips and Tricks

## Workaround for special characters

````html
<!-- HTML: -->
<div x-on:cdr.window="console.log('cdr from window received:', event.detail.id)">Hello</div>

<!-- JSX: -->
<div {...{
  'x-on:cdr.window': `console.log('cdr from window received:', event.detail.id)`
  }}
>Hello</div>
````

## Pass attributes from parent to child

### for a single target element

````JS
type TrAttrs = JSX.IntrinsicElements["tr"]

export const EvtPersondetailsRow = (
        props: { vm: EvtPersonDetailModel } & TrAttrs
) => (
<>
    <tr
            id={`row-${props.vm.id}`}
            {...props.attrs}
    >
        <td style="border-style: none"></td>
        <td style="border-style: none">{props.vm.firstName}</td>
    </tr>
</>
);

````

````JS
export const EvtPersonDetails = (props: { vm: OOBPersonDetailModel }) => (
        <>
            <EvtPersondetailsRow
                    vm={props.vm}
                    hx-trigger={`close-details-requested[event.detail.id === ${props.vm.id}] from:body`}
                    hx-get="/demo/event/person/{id}/row"
                    hx-vals='js:{id: event.detail.id}'
                    hx-target='closest tr'
                    hx-swap="outerHTML"
            >
            </EvtPersondetailsRow>
            <EvtPersondetailsCard vm={props.vm}/>
        </>
);
````

### for two target elements

````JS
type TrAttrs = JSX.IntrinsicElements["tr"]
type TdAttrs = JSX.IntrinsicElements["td"]

export const EvtPersondetailsRow = (
        {vm, children, tdAttrs, ...trAttrs}: {
            vm: EvtPersonDetailModel
            tdAttrs?: TdAttrs
            children: ComponentChildren
        } & TrAttrs
) => (
        <tr id={`row-${vm.id}`} {...trAttrs}>
            <td {...tdAttrs}>{vm.name}</td>
            {children}
        </tr>
)

````

````JS
<EvtPersondetailsRow
  vm={props.vm}
  hx-get="/demo/event/person/{id}/row"
  tdAttrs={{ class: "highlight", "hx-target": "this" }}
>
````
