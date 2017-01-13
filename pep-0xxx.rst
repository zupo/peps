PEP: XXX
Title: TLS Abstract Base Classes
Version: $Revision$
Last-Modified: $Date$
Author: Cory Benfield <cory@lukasa.co.uk>
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 17-Oct-2016
Python-Version: 3.7
Post-History: 30-Aug-2002


Abstract
========

This PEP would define a standard TLS interface in the form of a collection of
abstract base classes. This interface would allow Python implementations and
third-party libraries to provide bindings to TLS libraries other than OpenSSL
that can be used by tools that expect the interface provided by the Python
standard library, with the goal of reducing the dependence of the Python
ecosystem on OpenSSL.


Rationale
=========

In the 21st century it has become increasingly clear that robust and
user-friendly TLS support is an extremely important part of the ecosystem of
any popular programming language. For most of its lifetime, this role in the
Python ecosystem has primarily been served by the `ssl module`_, which provides
a Python API to the `OpenSSL library`_.

Because the ``ssl`` module is distributed with the Python standard library, it
has become the overwhelmingly most-popular method for handling TLS in Python.
An extraordinary majority of Python libraries, both in the standard library and
on the Python Package Index, rely on the ``ssl`` module for their TLS
connectivity.

Unfortunately, the preeminence of the ``ssl`` module has had a number of
unforseen side-effects that have had the effect of tying the entire Python
ecosystem tightly to OpenSSL. This has forced Python users to use OpenSSL even
in situations where it may provide a worse user experience than alternative TLS
implementations, which imposes a cognitive burden and makes it hard to provide
"platform-native" experiences.


Problems
--------

The fact that the ``ssl`` module is build into the standard library has meant
that all standard-library Python networking libraries are entirely reliant on
the OpenSSL that the Python implementation has been linked against. This
leads to the following issues:

* It is difficult to take advantage of new, higher-security TLS without
  recompiling Python to get a new OpenSSL. While there are third-party bindings
  to OpenSSL (e.g. `pyOpenSSL`_), these need to be shimmed into a format that
  the standard library understands, forcing projects that want to use them to
  maintain substantial compatibility layers.

* For Windows distributions of Python, they need to be shipped with a copy of
  OpenSSL. This puts the CPython development team in the position of being
  OpenSSL redistributors, potentially needing to ship security updates to the
  Windows Python distributions when OpenSSL vulnerabilities are released.

* For macOS distributions of Python, they need either to be shipped with a copy
  of OpenSSL or linked against the system OpenSSL library. Apple has formally
  deprecated linking against the system OpenSSL library, and even if they had
  not, that library version has been unsupported by upstream for nearly one
  year as of the time of writing. The CPython development team has started
  shipping newer OpenSSLs with the Python available from python.org, but this
  has the same problem as with Windows.

* Many systems, including but not limited to Windows and macOS, do not make
  their system certificate stores available to OpenSSL. This forces users to
  either obtain their trust roots from elsewhere (e.g. `certifi`_) or to
  attempt to export their system trust stores in some form.

  Relying on `certifi`_ is less than ideal, as most system administrators do
  not expect to receive security-critical software updates from PyPI.
  Additionally, it is not easy to extend the `certifi`_ trust bundle to include
  custom roots, or to centrally manage trust using the `certifi`_ model.

  Even in situations where the system certificate stores are made available to
  OpenSSL in some form, the experience is still sub-standard, as OpenSSL will
  perform different validation checks than the platform-native TLS
  implementation. This can lead to users experiencing different behaviour on
  their browsers or other platform-native tools than they experience in Python,
  with little or no recourse to resolve the problem.

* Users may wish to integrate with TLS libraries other than OpenSSL for many
  other reasons, such as OpenSSL missing features (e.g. TLS 1.3 support), or
  because OpenSSL is simply too large and unweildy for the platform (e.g. for
  embedded Python). Those users are left with the requirement to use
  third-party networking libraries that can interact with their preferred TLS
  library or to shim their preferred library into the OpenSSL-specific ``ssl``
  module API.

Additionally, the ``ssl`` module as implemented today limits the ability of
CPython itself to add support for alternative TLS backends, or remove OpenSSL
support entirely, should either of these become necessary or useful. The
``ssl`` module exposes too many OpenSSL-specific function calls and features to
easily map to an alternative TLS backend.


Proposal
========

This PEP proposes to introduce a few new Abstract Base Classes in Python 3.7 to
provide TLS functionality that is not so strongly tied to OpenSSL. It also
proposes to update standard library modules to use only the interface exposed
by these abstract base classes wherever possible. There are three goals here:

1. To provide a common API surface for both core and third-party developers to
   target their TLS implementations to. This allows TLS developers to provide
   interfaces that can be used by most Python code, and allows network
   developers to have an interface that they can target that will work with a
   wide range of TLS implementations.
2. To provide an API that has few or no OpenSSL-specific concepts leak through.
   The ``ssl`` module today has a number of warts caused by leaking OpenSSL
   concepts through to the API: the new ABCs would remove those specific
   concepts.
3. To provide a path for the core development team to make OpenSSL one of many
   possible TLS backends, rather than requiring that it be present on a system
   in order for Python to have TLS support.

The proposed interface is laid out below.


Abstract Base Classes
---------------------

There are several interfaces that require standardisation. Those interfaces
are:

1. Configuring TLS, currently implemented by the `SSLContext`_ class in the
   ``ssl`` module.
2. Wrapping a socket object, currently implemented by the `SSLSocket`_ class
   in the ``ssl`` module.
3. Providing an in-memory buffer for doing in-memory encryption or decryption
   with no actual I/O (necessary for asynchronous I/O models), currently
   implemented by the `SSLObject`_ class in the ``ssl`` module.
4. Specifying TLS cipher suites. There is currently no code for doing this in
   the standard library: instead, the standard library uses OpenSSL cipher
   suite strings.
5. Specifying application-layer protocols that can be negotiated during the
   TLS handshake.
6. Specifying TLS versions.
7. Reporting errors to the caller, currently implemented by the `SSLError`_
   class in the ``ssl`` module.
8. Specifying certificates to load, either as client or server certificates.
9. Specifying which trust database should be used to validate certificates
   presented by a remote peer.

While it is technically possible to define (2) in terms of (3), for the sake of
simplicity it is easier to define these as two separate ABCs. Implementations
are of course free to implement the concrete subclasses however they see fit.

Obviously, (4) doesn't require an abstract base class: instead, it requires a
richer API for configuring supported cipher suites that can be easily updated
with supported cipher suites for different implementations.

(8) is a thorny problem, becuase in an ideal world the private keys associated
with these certificates would never end up in-memory in the Python process
(that is, the TLS library would collaborate with a Hardware Security Module
(HSM) to provide the private key in such a way that it cannot be extracted from
process memory). Thus, we need to provide an extensible model of providing
certificates that allows concrete implementations the ability to provide this
higher level of security, while also allowing a lower bar for those
implementations that cannot. This lower bar would be the same as the status
quo: that is, the certificate may be loaded from an in-memory buffer or from a
file on disk.

(9) also represents an issue because different TLS implementations vary wildly
in how they allow users to select trust stores. Some implementations have
specific trust store formats that only they can use (such as the OpenSSL CA
directory format that is created by ``c_rehash``), and others may not allow you
to specify a trust store that does not include their default trust store.

For this reason, we need to provide a model that assumes very little about the
form that trust stores take. The "Trust Store" section below goes into more
detail about how this is achieved.

Context
~~~~~~~

The ``Context`` abstract base class defines an object that allows configuration
of TLS. It can be thought of as a factory for ``TLSWrappedSocket`` and
``TLSWrappedBuffer`` objects.

The ``Context`` abstract base class has the following class definition::

    TLSBufferObject = Union[TLSWrappedSocket, TLSWrappedBuffer]
    ServerNameCallback = Callable[[TLSBufferObject, Optional[str], Context], Any]

    class _BaseContext(metaclass=ABCMeta):

        @property
        @abstractmethod
        def validate_certificates(self) -> bool:
            """
            Whether to validate the TLS certificates. This switch operates at a
            very broad scope: either validation is enabled, in which case all
            forms of validation are performed including hostname validation if
            possible, or validation is disabled, in which case no validation is
            performed.

            Not all backends support having their certificate validation
            disabled. If a backend does not support having their certificate
            validation disabled, attempting to set this property to ``False``
            will throw a ``TLSError``.
            """

        @validate_certificates.setter
        @abstractmethod
        def validate_certificates(self, val: bool) -> None:
          pass

        @abstractmethod
        def register_certificates(self,
                                  certificates: List[Certificate],
                                  key: PrivateKey) -> None:
            """
            Loads a certificate, a number of intermediate certificates, and the
            corresponding private key. These certificates will be offered to
            the remote peer during the handshake if required.

            The ``certificates`` argument must be a list of Certificate objects
            as detailed below. The first Certificate must be the leaf
            certificate. All subsequent certificates will be offered as
            intermediate additional certificates.

            The ``key`` argument must contain the private key associated with
            the leaf certificate.
            """

        @abstractmethod
        def set_ciphers(self, ciphers: List[Ciphers]) -> None:
            """
            Set the available ciphers for TLS connections created with this
            context. ``ciphers`` should be a list of ciphers from the
            ``Cipher`` registry. If none of the ``ciphers`` provided to this
            object are supported or available, a ``TLSError`` will be raised.
            """

        @abstractmethod
        def set_inner_protocols(self, protocols: List[Union[NextProtocol, bytes]]) -> None:
            """
            Specify which protocols the socket should advertise as supported
            during the TLS handshake. This may be advertised using either or
            both of ALPN or NPN.

            ``protocols`` should be a list of acceptable protocols in the form
            of ``NextProtocol`` objects, such as
            ``[NextProtocol.H2, NextProtocol.HTTP1]``, ordered by preference.
            The selection of the protocol will happen during the handshake,
            and will use whatever protocol negotiation mechanisms are available
            and supported by both peers. If the ``NextProtocol`` enum doesn't
            contain a value for the protocol you'd like to negotiate, a byte
            string can also be passed directly instead.

            If the TLS implementation doesn't support protocol negotiation,
            this method will raise ``NotImplementedError``.
            """

        @abstractmethod
        def set_version_range(self, lower_bound=None: Optional[TLSVersion],
                              upper_bound=None: Optional[TLSVersion]) -> None:
            """
            Set the minumum and maximum versions of TLS that should be allowed
            on TLS connections made by this context.

            If present, ``lower_bound`` will set the lowest acceptable TLS
            version. If present, ``upper_bound`` will set the highest
            acceptable TLS version. If either argument is ``None``, this will
            leave that bound unchanged.
            """

        @abstractmethod
        def use_trust_store(self, store: TrustStore) -> None:
            """
            Configures the Context to use a given trust store to validate
            certificates. If not called, the default trust store will be used.
            """


    class ClientContext(_BaseContext):
        @abstractmethod
        def wrap_socket(self, socket: socket.socket,
                        auto_handshake=True: bool,
                        server_hostname=None: Optional[str]) -> TLSWrappedSocket:
            """
            Wrap an existing Python socket object ``socket`` and return a
            ``TLSWrappedSocket`` object. ``socket`` must be a ``SOCK_STREAM``
            socket: all other socket types are unsupported.

            The returned SSL socket is tied to the context, its settings and
            certificates.

            The parameter ``auto_handshake`` specifies whether to do the SSL
            handshake automatically after doing a ``socket.connect()``, or
            whether the application program will call it explicitly, by
            invoking the ``TLSWrappedSocket.do_handshake()`` method. Calling
            ``TLSWrappedSocket.do_handshake()`` explicitly gives the program
            control over the blocking behavior of the socket I/O involved in
            the handshake.

            The optional parameter ``server_hostname`` specifies the hostname
            of the service which we are connecting to. This allows a single
            server to host multiple SSL-based services with distinct
            certificates, quite similarly to HTTP virtual hosts.
            """

        @abstractmethod
        def wrap_buffers(self, incoming: Any, outgoing: Any,
                         server_hostname=None: Optional[str]) -> TLSWrappedBuffer:
            """
            Wrap a pair of buffer objects (``incoming`` and ``outgoing``) to
            create an in-memory stream for TLS. The SSL routines will read data
            from ``incoming`` and decrypt it, and write encrypted data to
            ``outgoing``.

            The ``server_hostname`` parameter has the same meaning as in
            ``wrap_socket``.
            """


    class ServerContext(_BaseContext):
        @abstractmethod
        def wrap_socket(self, socket: socket.socket,
                        auto_handshake=True: bool) -> TLSWrappedSocket:
            """
            Wrap an existing Python socket object ``socket`` and return a
            ``TLSWrappedSocket`` object. ``socket`` must be a ``SOCK_STREAM``
            socket: all other socket types are unsupported.

            The returned SSL socket is tied to the context, its settings and
            certificates.

            The parameter ``auto_handshake`` specifies whether to do the SSL
            handshake automatically after doing a ``socket.connect()``, or
            whether the application program will call it explicitly, by
            invoking the ``TLSWrappedSocket.do_handshake()`` method. Calling
            ``TLSWrappedSocket.do_handshake()`` explicitly gives the program
            control over the blocking behavior of the socket I/O involved in
            the handshake.
            """

        @abstractmethod
        def wrap_buffers(self, incoming: Any, outgoing: Any) -> TLSWrappedBuffer:
            """
            Wrap a pair of buffer objects (``incoming`` and ``outgoing``) to
            create an in-memory stream for TLS. The SSL routines will read data
            from ``incoming`` and decrypt it, and write encrypted data to
            ``outgoing``.
            """

        @abstractmethod
        def set_sni_callback(self, callback: Optional[ServerNameCallback]) -> None:
            """
            Register a callback function that will be called after the TLS
            Client Hello handshake message has been received by the TLS server
            when the TLS client specifies a server name indication.

            Only one callback can be set per ``Context``. If ``callback`` is
            ``None`` then the callback is disabled. Calling this function a
            subsequent time will disable the previously registered callback.

            The ``callback`` function will be called with three arguments: the
            first will be the ``TLSBufferObject`` for the connection; the
            second will be a string that represents the server name that the
            client is intending to communicate (or ``None`` if the TLS Client
            Hello does not contain a server name); and the third argument will
            be the original ``Context``. The server name argument will be the
            IDNA *decoded* server name.

            The ``callback`` must return ``None`` to allow negotiation to
            continue. Other return values signal errors. Attempting to control
            what error is signaled by the underlying TLS implementation is not
            specified in this API, but is up to the concrete implementation to
            handle.
            """


Socket
~~~~~~

The socket-wrapper ABC will be defined by the ``TLSWrappedSocket`` ABC, which
has the following definition::

    class TLSWrappedSocket(metaclass=ABCMeta):
        # The various socket methods all must be implemented. Their definitions
        # have been elided from this class defintion in the PEP because they
        # aren't instructive.
        @abstractmethod
        def do_handshake(self) -> None:
            """
            Performs the TLS handshake. Also performs certificate validation
            and hostname verification.
            """

        @abstractmethod
        def cipher(self) -> Optional[Cipher]:
            """
            Returns the Cipher entry for the cipher that has been negotiated on
            the connection. If no connection has been negotiated, returns
            ``None``.
            """

        @abstractmethod
        def negotiated_protocol(self) -> Optional[Union[NextProtocol, bytes]]:
            """
            Returns the protocol that was selected during the TLS handshake.
            This selection may have been made using ALPN, NPN, or some future
            negotiation mechanism.

            If the negotiated protocol is one of the protocols defined in the
            ``NextProtocol`` enum, the value from that enum will be returned.
            Otherwise, the raw bytestring of the negotiated protocol will be
            returned.

            If ``Context.set_inner_protocols()`` was not called, if the other
            party does not support protocol negotiation, if this socket does
            not support any of the peer's proposed protocols, or if the
            handshake has not happened yet, ``None`` is returned.
            """

        @property
        @abstractmethod
        def context(self) -> Context:
            """
            The ``Context`` object this socket is tied to.
            """

        @context.setter
        @abstractmethod
        def context(self, value: Context) -> None:
            """
            Set the value of the ``Context`` object this socket is tied to.
            This operation (changing the context) may not always be supported.
            """

        @abstractproperty
        def negotiated_tls_version(self) -> Optional[TLSVersion]:
            """
            The version of TLS that has been negotiated on this connection.
            """

        @abstractmethod
        def unwrap(self) -> socket.socket:
            """
            Cleanly terminate the TLS connection on this wrapped socket. Once
            called, this ``TLSWrappedSocket`` can no longer be used to transmit
            data. Returns the socket that was wrapped with TLS.
            """


Buffer
~~~~~~

The buffer-wrapper ABC will be defined by the ``TLSWrappedBuffer`` ABC, which
has the following definition::

    class TLSWrappedBuffer(metaclass=ABCMeta):
        @abstractmethod
        def read(self, amt=None: int) -> bytes:
            """
            Read up to ``amt`` bytes of data from the input buffer and return
            the result as a ``bytes`` instance. If ``amt`` is ``None``, will
            attempt to read until either EOF is reached or until further
            attempts to read would raise either ``WantReadError`` or
            ``WantWriteError``.

            Raise ``WantReadError`` or ``WantWriteError`` if there is
            insufficient data in either the input or output buffer and the
            operation would have caused data to be written or read.

            As at any time a re-negotiation is possible, a call to ``read()``
            can also cause write operations.
            """

        @abstractmethod
        def read_into(self, buffer: Any, amt=None: int) -> int:
            """
            Read up to ``amt`` bytes of data from the input buffer into
            ``buffer``, which must be an object that implements the buffer
            protocol. Returns the number of bytes read. If ``amt`` is ``None``,
            will attempt to read until either EOF is reached or until further
            attempts to read would raise either ``WantReadError`` or
            ``WantWriteError``, or until the buffer is full.

            Raises ``WantReadError`` or ``WantWriteError`` if there is
            insufficient data in either the input or output buffer and the
            operation would have caused data to be written or read.

            As at any time a re-negotiation is possible, a call to
            ``read_into()`` can also cause write operations.
            """

        @abstractmethod
        def write(self, buf: Any) -> int:
            """
            Write ``buf`` in encrypted form to the output buffer and return the
            number of bytes written. The ``buf`` argument must be an object
            supporting the buffer interface.

            Raise ``WantReadError`` or ``WantWriteError`` if there is
            insufficient data in either the input or output buffer and the
            operation would have caused data to be written or read.

            As at any time a re-negotiation is possible, a call to ``write()``
            can also cause read operations.
            """

        @abstractmethod
        def do_handshake(self) -> None:
            """
            Performs the TLS handshake. Also performs certificate validation
            and hostname verification.
            """

        @abstractmethod
        def cipher(self) -> Optional[Cipher]:
            """
            Returns the Cipher entry for the cipher that has been negotiated on
            the connection. If no connection has been negotiated, returns
            ``None``.
            """

        @abstractmethod
        def negotiated_protocol(self) -> Optional[Union[NextProtocol, bytes]]:
            """
            Returns the protocol that was selected during the TLS handshake.
            This selection may have been made using ALPN, NPN, or some future
            negotiation mechanism.

            If the negotiated protocol is one of the protocols defined in the
            ``NextProtocol`` enum, the value from that enum will be returned.
            Otherwise, the raw bytestring of the negotiated protocol will be
            returned.

            If ``Context.set_inner_protocols()`` was not called, if the other
            party does not support protocol negotiation, if this socket does
            not support any of the peer's proposed protocols, or if the
            handshake has not happened yet, ``None`` is returned.
            """

        @property
        @abstractmethod
        def context(self) -> Context:
            """
            The ``Context`` object this socket is tied to.
            """

        @context.setter
        @abstractmethod
        def context(self, value: Context) -> None:
            """
            Set the value of the ``Context`` object this socket is tied to.
            This operation (changing the context) may not always be supported.
            """

        @abstractproperty
        def negotiated_tls_version(self) -> Optional[TLSVersion]:
            """
            The version of TLS that has been negotiated on this connection.
            """

        @abstractmethod
        def shutdown(self) -> None:
            """
            Performs a clean TLS shut down. This should generally be used
            whenever possible to signal to the remote peer that the content is
            finished.
            """


Cipher Suites
~~~~~~~~~~~~~

Todo

Protocol Negotiation
~~~~~~~~~~~~~~~~~~~~

Both NPN and ALPN allow for protocol negotiation as part of the HTTP/2
handshake. While NPN and ALPN are, at their fundamental level, built on top of
bytestrings, string-based APIs are frequently problematic as they allow for
errors in typing that can be hard to detect.

For this reason, this module would define a type that protocol negotiation
implementations can pass and be passed. This type would wrap a bytestring to
allow for aliases for well-known protocols. This allows us to avoid the
problems inherent in typos for well-known protocols, while allowing the full
extensibility of the protocol negotiation layer if needed by letting users pass
byte strings directly.

::

    class NextProtocol(Enum):
        H2 = b'h2'
        H2C = b'h2c'
        HTTP1 = b'http/1.1'
        WEBRTC = b'webrtc'
        C_WEBRTC = b'c-webrtc'
        FTP = b'ftp'
        STUN = b'stun.nat-discovery'
        TURN = b'stun.turn'

TLS Versions
~~~~~~~~~~~~

It is often useful to be able to restrict the versions of TLS you're willing to
support. There are many security advantages in refusing to use old versions of
TLS, and some misbehaving servers will mishandle TLS clients advertising
support for newer versions.

The following enumerated type can be used to gate TLS versions. Forward-looking
applications should almost never set a maximum TLS version unless they
absolutely must, as a TLS backend that is newer than the Python that uses it
may support TLS versions that are not in this enumerated type.

Additionally, this enumerated type defines two additional flags that can always
be used to request either the lowest or highest TLS version supported by an
implementation.

::

    class TLSVersion(Enum):
        MINIMUM_SUPPORTED
        SSLv2
        SSLv3
        TLSv1
        TLSv1_1
        TLSv1_2
        TLSv1_3
        MAXIMUM_SUPPORTED


Errors
~~~~~~

This module would define three base classes for use with error handling. Unlike
the other classes defined here, these classes are not *abstract*, as they have
no behaviour. They exist simply to signal certain common behaviours. Backends
should subclass these exceptions in their own packages, but needn't define any
behaviour for them.

In general, concrete implementations should subclass these exceptions rather
than throw them directly. This makes it moderately easier to determine which
concrete TLS implementation is in use during debugging of unexpected errors.
However, this is not mandatory.

The definitions of the errors are below::

    class TLSError(Exception):
        """
        The base exception for all TLS related errors from any backend.
        Catching this error should be sufficient to catch *all* TLS errors,
        regardless of what backend is used.
        """

    class WantWriteError(TLSError):
        """
        A special signaling exception used only when non-blocking or
        buffer-only I/O is used. This error signals that the requested
        operation cannot complete until more data is written to the network,
        or until the output buffer is drained.
        """

    class WantReadError(TLSError):
        """
        A special signaling exception used only when non-blocking or
        buffer-only I/O is used. This error signals that the requested
        operation cannot complete until more data is read from the network, or
        until more data is available in the input buffer.
        """


Certificates
~~~~~~~~~~~~

This module would define an abstract X509 certificate class. This class would
have almost no behaviour, as the goal of this module is not to provide all
possible relevant cryptographic functionality that could be provided by X509
certificates. Instead, all we need is the ability to signal the source of a
certificate to a concrete implementation.

For that reason, this certificate implementation defines only constructors. In
essence, the certificate object in this module could be as abstract as a handle
that can be used to locate a specific certificate.

Concrete implementations may choose to provide alternative constructors, e.g.
to load certificates from HSMs. If a common interface emerges for doing this,
this module may be updated to provide a standard constructor for this use-case
as well.

    class Certificate(metaclass=ABCMeta):
        @abstractclassmethod
        def from_buffer(self, buffer: bytes) -> Certificate:
            """
            Creates a Certificate object from a byte buffer. This byte buffer
            may be either PEM-encoded or DER-encoded. If the buffer is PEM
            encoded it *must* begin with the standard PEM preamble (a series of
            dashes followed by the ASCII bytes "BEGIN CERTIFICATE" and another
            series of dashes). In the absence of that preamble, the
            implementation may assume that the certificate is DER-encoded
            instead.
            """

        @abstractclassmethod
        def from_file(self, path: Union[pathlib.Path, AnyStr]) -> Certificate:
            """
            Creates a Certificate object from a file on disk. This method may
            be a convenience method that wraps ``open`` and ``from_buffer``,
            but some TLS implementations may be able to provide more-secure or
            faster methods of loading certificates that do not involve Python
            code.
            """


Private Keys
~~~~~~~~~~~~

This module would define an abstract private key class. Much like the
Certificate class, this class has almost no behaviour in order to give as much
freedom as possible to the concrete implementations to treat keys carefully.

This class has all the caveats of the ``Certificate`` class.

    class PrivateKey(metaclass=ABCMeta):
        @abstractclassmethod
        def from_buffer(self,
                        buffer: bytes,
                        password=None: Optional[Union[Callable[[], Union[bytes, bytearray]], bytes, bytearray]) -> PrivateKey:
            """
            Creates a PrivateKey object from a byte buffer. This byte buffer
            may be either PEM-encoded or DER-encoded. If the buffer is PEM
            encoded it *must* begin with the standard PEM preamble (a series of
            dashes followed by the ASCII bytes "BEGIN", the key type, and
            another series of dashes). In the absence of that preamble, the
            implementation may assume that the certificate is DER-encoded
            instead.

            The key may additionally be encrypted. If it is, the ``password``
            argument can be used to decrypt the key. The ``password`` argument
            may be a function to call to get the password for decrypting the
            private key. It will only be called if the private key is encrypted
            and a password is necessary. It will be called with no arguments,
            and it should return either bytes or bytearray containing the
            password. Alternatively a bytes, or bytearray value may be supplied
            directly as the password argument. It will be ignored if the
            private key is not encrypted and no password is needed.
            """

        @abstractclassmethod
        def from_file(self,
                      path: Union[pathlib.Path, bytes, str],
                      password=None: Optional[Union[Callable[[], Union[bytes, bytearray]], bytes, bytearray]) -> PrivateKey:
            """
            Creates a PrivateKey object from a file on disk. This method may
            be a convenience method that wraps ``open`` and ``from_buffer``,
            but some TLS implementations may be able to provide more-secure or
            faster methods of loading certificates that do not involve Python
            code.

            The ``password`` parameter behaves exactly as the equivalent
            parameter on ``from_buffer``.
            """


Trust Store
~~~~~~~~~~~

As discussed above, loading a trust store represents an issue because different
TLS implementations vary wildly in how they allow users to select trust stores.
For this reason, we need to provide a model that assumes very little about the
form that trust stores take.

This problem is the same as the one that the Certificate and PrivateKey types
need to solve. For this reason, we use the exact same model, by creating an
opaque type that can encapsulate the various means that TLS backends may open
a trust store.

A given TLS implementation is not required to implement all of the
constructors. However, it is strongly recommended that a given TLS
implementation provide the ``system`` constructor if at all possible, as this
is the most common validation trust store that is used. Concrete
implementations may also add their own constructors.

    class TrustStore(metaclass=ABCMeta):
        @abstractclassmethod
        def system(cls) -> TrustStore:
            """
            Returns a TrustStore object that represents the system trust
            database.
            """

        @abstractclassmethod
        def from_pem_file(cls, path: Union[pathlib.Path, bytes, str]) -> TrustStore:
            """
            Initializes a trust store from a single file full of PEMs.
            """


Changes to the Standard Library
===============================

The portions of the standard library that interact with TLS should be revised
to use these ABCs. This will allow them to function with other TLS backends.
This includes the following modules:

- asyncio
- ftplib
- http.client
- imaplib
- nntplib
- poplib
- smtplib


Future
======

Major future TLS features may require revisions of these ABCs. These revisions
should be made cautiously: many backends may not be able to move forward
swiftly, and will be invalidated by changes in these ABCs. This is acceptable,
but wherever possible features that are specific to individual implementations
should not be added to the ABCs. The ABCs should restrict themselves to
high-level descriptions of IETF-specified features.


ToDo
====

* Consider adding a new parameter (``valid_subjects``?) to ``wrap_socket`` and
  ``wrap_buffers`` that specifies in a *typed* manner what kind of entries in
  the SAN field are acceptable. This would break the union between SNI and
  cert validation, which may be a good thing (you can't SNI an IP address, but
  you can validate a cert with one if you want).
* It's annoying that there's no type signature for fileobj. Do I really have to
  define one as part of this PEP? Otherwise, how do I define the types of the
  arguments to ``wrap_buffers``?
* Do we need ways to control hostname validation?
* Do we need to support getpeercert? Should we always return DER instead of the
  weird semi-structured thing?
* How do we load certs from locations on disk? What about HSMs?
* How do we signal to load certs from the OS? What happens if an implementation
  doesn't let you *not* load those certs?


References
==========

.. _ssl module: https://docs.python.org/3/library/ssl.html
.. _OpenSSL Library: https://www.openssl.org/
.. _PyOpenSSL: https://pypi.org/project/pyOpenSSL/
.. _certifi: https://pypi.org/project/certifi/
.. _SSLContext: https://docs.python.org/3/library/ssl.html#ssl.SSLContext
.. _SSLSocket: https://docs.python.org/3/library/ssl.html#ssl.SSLSocket
.. _SSLObject: https://docs.python.org/3/library/ssl.html#ssl.SSLObject
.. _SSLError: https://docs.python.org/3/library/ssl.html#ssl.SSLError


Copyright
=========

This document has been placed in the public domain.



..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:
