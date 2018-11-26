# mitmengine

The goal of this project is to allow for accurate detection of HTTPS interception and robust TLS fingerprinting. 
This project is based off of [The Security Impact of HTTPS Interception](https://zakird.com/papers/https_interception.pdf), and started as a port to Go of [their processing scripts and fingerprints](https://github.com/zakird/tlsfingerprints).

## Signature and Fingerprints
In this project, fingerprints map to concrete instantiations of an object, while signatures can represent multiple objects. We use this convention because a fingerprint is usually an inherent property of an object, while a signature can be chosen. In the same way, an actual client request seen by a server would have a fingerprint, while the software generating the request can choose it's own signature (e.g., by choosing which cipher suites it supports).

### Client Request
A client request fingerprint is derived from a client request to a server, and contains both TLS and HTTP features. A client request signature represents all of the possible fingerprints that a piece of software can generate. The aim is to make each signature specific enough that it can uniquely identify a piece of software.

### User Agent
A user agent signature represent a set of user agents generated by a browser. A user agent signature for a browser allows for a range of browser versions, and allows for specifying the OS name, OS platform, OS version range, and device type for creating more fine-grained signatures.

### Browser
A browser signature contains both a user agent signature and a client request signature. This allows for a signature to represent all of the possible fingerprints generated by Chrome 31-38 on Windows 10, for example.

### MITM
A MITM signature contains a client request signature along with additional details about the MITM software, including a security grade which can be affected by factors outside of the client request, such as whether or not the software validates certificates.

## MITM Detection Methodology
We consider an HTTPS connection to be intercepted when there is a mismatch
between the expected client request signature corresponding to the browser
identified by the user agent, and the actual client request fingerprint of the
request.

### False positives
If a signature is inaccurate or outdated for a given piece of client software,
it is possible that the signature will falsely flag a connection as being
intercepted.

### False negatives
If a proxy closely mimics the request of the client, then we may not expect to
detect a mismatch. If the browser signatures are overly broad, we will also
fail to detect interception.

## Testing
To test, run ```make test``` and to see code coverage, run ```make cover```.

## Godoc
Run ```make godoc``` or ```PKG=<sub-package> make godoc``` to generate godoc for mitmengine or any of the sub-packages.

## API
First, a user must create a `mitmengine.Config` struct to pass into `mitmengine.NewProcessor`. A `mitmengine.Config` 
struct can either specify filenames of files containing browser fingerprints, MITM fingerprints, and man-in-the-middle 
headers. Alternatively, it can also specify a configuration file for reading the previously mentioned files from any 
other source; right now, mitmengine supports reading these files from Amazon S3 client-compatible databases (including 
Amazon S3 and Ceph). Additional file readers for databases (which we call "loaders") can be defined in the `loaders` 
package, and as long as new loaders extend the Loader interface, they should work with the rest of mitmengine out of the 
box. We added support for additional fingerprint and bad header sources in the case mitmengine is run as a daemon and 
you want to have it periodically update the fingerprint and bad header files it uses to analyze traffic.

The intended entrypoint to the mitmengine package is through the `Processor.Check` function, which takes a user agent and client request fingerprint, and returns a mitm detection report. Additional API functions will be added in the future to allow for adding new signatures to a running process, for example.
