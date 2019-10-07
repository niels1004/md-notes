# Case definition expressions for bone-supported guide
In order to move CT Preparation and later Implant Studio module to Triart 2.0 and New API, it is a requirement that we can create case definition expressions that discriminate between surgical guide on teeth, surgical guide on gums and surgical guide on bone. Currently, it is possible to discriminate between the first two by using `IsJawEdentulous`. But with the introduction of bone-supported guides IS supports two different kinds of (or at least two different workflows for designing) guides for edentulous patients.

## Proposal A
Discriminate between guide types for edentulous patient by introducing a new `ProductType` value.

Good:
* Most backward compatible.
* Potentially enables a new workflow with `!IsJawEdentulous` in combination with bone-supported guide product.

Cons:
* Lacks some consistency, i.e. tooth-supported and denture-based guides have same product type while bone-supported has a distinct type.

## Proposal B
Deprecate `IsJawEdentulous` and introduce separate `ProductType` values for denture based guide and bone-supported guide.

Pros:
* Models usage more closely. I.e. the practitioner orders a specific treatment, and it is not the software that infers the treatment from described symptoms in combination with a product description.

## Dental System compatibility
We can either require that Dental System is updated to support the new product types.

Alternatively, we can required 3DD to perform some reversable conversion to existing product types when exporting, e.g. by encoding some information in custom fields, and restore the product types when importing.