# Architecture decision records

An [architecture
decision](https://cloud.google.com/architecture/architecture-decision-records)
is a software design choice that evaluates:

-   a functional requirement (features).
-   a non-functional requirement (technologies, methodologies, libraries).

The purpose is to understand the reasons behind the current architecture, so
they can be carried-on or re-visited in the future.

## Initial idea

### **Problem Summary**

Design a Dart library to manage downloading of remote content based on a
flexible "resource key" system. The library must support multiple protocols
(e.g., HTTPS, AWS S3), handle concurrent downloads responsibly, provide
cache-like storage management with lifecycle tracking, and offer customizable
behavior at key processing stages.

***

### **Use Cases**

-   Fetch JSON payloads using HTTPS from known API endpoints.
-   Download binary media files (e.g., images) from public S3 buckets using
    ARNs as resource keys.
-   Validate and cache content fetched via JWTs with embedded expiration
    metadata.
-   Use resource keys in varying formats (URN, ARN, JWT) and map them to
    protocol-specific download parameters.
-   Trigger UI updates or internal events upon download completion or
    failure.
-   Store downloaded files in a secure directory tree with scoped access.
-   Provide a queue mechanism for batch downloads with controlled
    concurrency.
-   Automatically discard expired or obsolete cache files to reclaim
    storage.
-   Validate that a downloaded payload conforms to a required schema before
    marking as ready.

***

### **Edge Cases**

-   Resource key format is unrecognized or unsupported.
-   A JWT resource key has already expired at time of reception.
-   A download fails due to a transient network issue—should retry after
    backoff.
-   Simultaneous requests for the same resource key.
-   Local storage reaches capacity mid-download.
-   Malformed payload passes download but fails validation.
-   Resource key maps to different versions over time—cache must reflect
    version.
-   Content type mismatch (e.g., expected image but received HTML).

***

### **Limitations and Exclusions**

-   Do not implement actual HTTP or S3 download logic—delegate to pluggable
    download handlers.
-   Do not persist the download queue or cache table across app restarts.
-   Do not manage encryption or secure authentication—handled outside
    scope.
-   Do not assume universal resource key structure—must be user-defined or
    user-extensible.
-   Do not support uploading or two-way synchronization—download-only
    manager.
-   Do not implement UI-level features (progress bars, widgets, etc).

***

### **Suggested Pull API (Functions)**

-   `queueMessage(resourceKey)`: Accepts a resource key and queues it for
    download.
-   `isReady(resourceKey)`: Returns whether content is ready for access.
-   `getContent(resourceKey)`: Returns parsed object if applicable.
-   `getLocalFileUrl(resourceKey)`: Returns local path to file.
-   `getStatus(resourceKey)`: Returns download status (queued, in progress,
    ready, failed).
-   `evict(resourceKey)`: Forces removal from cache.
-   `purgeExpired()`: Removes all expired or discarded content.
-   `setDownloadHandler(protocol, handlerFn)`: Registers protocol-specific
    downloader.
-   `setValidationHandler(handlerFn)`: Registers validation function
    post-download.
-   `setResourceKeyMapper(mapperFn)`: Maps resource key to storage and
    download metadata.

***

### **Suggested Push API (Events / Hooks)**

-   `onQueued(resourceKey)`
-   `onStart(resourceKey)`
-   `onProgress(resourceKey, bytesDownloaded, totalBytes)`
-   `onSuccess(resourceKey)`
-   `onFailure(resourceKey, reason)`
-   `onValidateSuccess(resourceKey)`
-   `onValidateFailure(resourceKey, reason)`
-   `onEvicted(resourceKey)`
-   `onExpired(resourceKey)`
