# Derive data from objects rather than primitives

A common problem that developers have when learning a new codebase
is finding where data *comes from*. If we structure methods and
functions in a way that prefers objects rather than primitives 
(or intermediary data formats) when finding derived data, this becomes
less burdensome for someone to learn, and easier to refactor.

## Examples of primitive methods

```php
// The method in a service
public function isDocumentDraft(string status): bool {
    return status === 'draft' || status === 'under_review';
}

// Is called later on after some data has been placed into
// an intermediary array data format:
if ($this->isDocumentDraft($importantMetadata['docStatus'])) {
    // Something else happens
}

// But the developer still doesn't know where this data is retrieved
// from - what is the source of the document status?
// We know where the document is coming from:
$document = $this->getDocument($id);

// ... sometime later
$importantMetadata = [
    'docStatus' => $document->getWorkflow()->getState()->getLabel(),
    // .. some other pieces of metadata
];
```

So when a developer has been tasked with finding why there is an issue with 
the workflow status for a document, they need to search the codebase
to find the source of the data.

Whilst intermediate data structures might be useful buckets to
store data for later easy access, it makes life harder when it comes
to refactoring or support as the source of the data is now opaque.

In the preceding example we are deriving whether a document is `draft` or not, 
and the determination of this status is based under a business rule:

*The document must have a state label of 'draft' or 'under_review' to be
classified as 'in draft'*

This begs whether the business rule is correct, however being able to
find this rule easily in the code will aid future discussions, and
mean that if the rule does change, that the amend is only needed in one
place rather than throughout the system to adjust the intermediary
data structures.

Consider changing the method to:

```php
public function isDocumentDraft(Document $document): bool {
    $status = $document->getWorkflow()->getState()->getLabel();
    return status === 'draft' || status === 'under_review';
}
```

This tiny change has meant that all the information about the business
rule is within a single method. There are no extra steps necessary
to extract a string (or know which string?!) or to know if the 
method is reusable for objects other than a document.

Static code analysis can now be run when we refactor to change the 
interface, and the rest of the application no longer cares if
we change the underlying business rule for determining if a document
is draft. E.g. changing the signature to this is an easy job with
an IDE:

```php
public function isDocumentDraft(Document $document): bool {
    $statusType = $document->getWorkflow()->getState()->getStatusType();
    return $statusType === StatusTypeEnum::Unpublished;
}
```

And extension for different object types, or an interface, easily
allows explanatory reuse:

```php
public function isDocumentDraft(WorkflowStateInterface $workflowObject): bool {
    $statusType = $workflowObject->getWorkflow()->getState()->getStatusType();
    return $statusType === StatusTypeEnum::Unpublished;
}
```

Much more reusable and self contained with its logic - support is easier, as
are changes to the underlying system.

## Conclusion

Please consider not using primitives as method signatures.
