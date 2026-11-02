# A small Jamstack photo app: keep the page static and the job state on the server

A photo tool looks like a simple web page: choose an image, pick a style, and wait for a result. The interesting architecture begins after that last click. A model request can take time, cost money, or return an uncertain response. None of those facts belongs only in a browser button's state.

This note uses AI Photo Styler as a concrete example. I work on the product. It has a small set of 1980s portrait styles, a comparison slider, and a JPEG download. The useful lesson is the boundary between the interface and the work behind it, rather than a particular model or framework.

## Start with a page, then add a job

The interface is authored as HTML, CSS, and browser JavaScript. A Cloudflare Worker serves those static assets and handles the application's requests. The interactive controls do not require a different server-rendered page for each style.

The Worker calls the external services and keeps job and quota records in D1. The browser receives progress and displays a usable result when the server has confirmed it.

```text
Static interface and browser interaction
                  |
           Application Worker
                  |
       Job record and quota reservation
                  |
           Input safety check
                  |
           Generation provider
                  |
       Output safety and image validation
                  |
          Confirmed result to browser
```

Calling this a static interface does not mean removing the backend. The page and the paid operation have different responsibilities. Keeping them separate makes it easier to change a layout without moving a provider key into the download bundle.

## Give each layer a clear job

| Layer | Responsibility |
| --- | --- |
| Browser | Validate the selected file for a useful early error, normalize the image, show progress, compare versions, and prepare the download. |
| Worker | Check the request again, own provider credentials, authorize result access, and decide whether processing may proceed. |
| Database | Keep the job's state and quota reservation across requests. |
| External services | Perform image generation and the input and output safety checks. |

Client validation helps the person using the tool. Server validation protects the operation. They are complementary checks: a visitor can bypass the interface and send a request directly.

## Reserve before spending

The server records a generation and reserves its allowance before asking providers to do work. This matters when several tabs arrive together. A disabled button in one tab cannot coordinate the others.

The reservation is also separate from ownership. Two people on the same Wi-Fi can share an allowance without gaining access to one another's job results. A quota identifier is not a result-access credential.

There is another state worth representing explicitly: unknown. If a paid generation might have reached the provider but its receipt is missing, the app should not quietly submit a second generation. It keeps that uncertainty distinct from a confirmed failure and tells the visitor that the photo is being checked. A confirmed failure can release the personal slot; an unresolved submission needs a different path.

## Success needs more than a provider response

The application checks the upload before generation and the generated output before completion. It also retrieves and validates the image before recording success. An unavailable safety check stops that path rather than showing an unchecked result.

This is useful beyond photo tools. A conversion job is not complete because a task was queued, and an export is not complete because a remote service returned a URL. Define success at the point where the application has a usable, authorized result.

## Keep retention claims narrow

Recent photos and favorites in this interface are held in the current browser when its storage is available. That is a convenience, not a cross-device account or a permanent backup. The interface offers a download so people can keep the images they choose.

External processing still happens. Browser-local history does not mean that uploaded images never leave the device, and an application's result-expiry policy is not proof of deletion from every provider. Explain those boundaries plainly instead of turning an architecture choice into an absolute privacy claim.

## A small review checklist

Before connecting another paid operation to a static page, ask:

- Can provider credentials stay entirely on the server?
- What record survives a refresh or a second tab?
- What proves success, and what happens when the result is uncertain?
- Who can read the result, independently of who shares a quota?
- Which checks must finish before a result becomes visible?
- What should the visitor download rather than trust to browser history?

The example interface is [AI Photo Styler](https://aiphotostyler.com/). This article describes its design, not a performance benchmark, an independent endorsement, or a claim that this is the only suitable stack. No application source, credentials, or customer images are included.
