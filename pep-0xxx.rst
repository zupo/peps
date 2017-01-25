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
unforeseen side-effects that have had the effect of tying the entire Python
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
4. Applying TLS configuration to the wrapping objects in (2) and (3). Currently
   this is also implemented by the `SSLContext`_ class in the ``ssl`` module.
5. Specifying TLS cipher suites. There is currently no code for doing this in
   the standard library: instead, the standard library uses OpenSSL cipher
   suite strings.
6. Specifying application-layer protocols that can be negotiated during the
   TLS handshake.
7. Specifying TLS versions.
8. Reporting errors to the caller, currently implemented by the `SSLError`_
   class in the ``ssl`` module.
9. Specifying certificates to load, either as client or server certificates.
10. Specifying which trust database should be used to validate certificates
    presented by a remote peer.
11. Finding a way to get hold of these interfaces at run time.

While it is technically possible to define (2) in terms of (3), for the sake of
simplicity it is easier to define these as two separate ABCs. Implementations
are of course free to implement the concrete subclasses however they see fit.

Obviously, (5) doesn't require an abstract base class: instead, it requires a
richer API for configuring supported cipher suites that can be easily updated
with supported cipher suites for different implementations.

(9) is a thorny problem, becuase in an ideal world the private keys associated
with these certificates would never end up in-memory in the Python process
(that is, the TLS library would collaborate with a Hardware Security Module
(HSM) to provide the private key in such a way that it cannot be extracted from
process memory). Thus, we need to provide an extensible model of providing
certificates that allows concrete implementations the ability to provide this
higher level of security, while also allowing a lower bar for those
implementations that cannot. This lower bar would be the same as the status
quo: that is, the certificate may be loaded from an in-memory buffer or from a
file on disk.

(10) also represents an issue because different TLS implementations vary wildly
in how they allow users to select trust stores. Some implementations have
specific trust store formats that only they can use (such as the OpenSSL CA
directory format that is created by ``c_rehash``), and others may not allow you
to specify a trust store that does not include their default trust store.

For this reason, we need to provide a model that assumes very little about the
form that trust stores take. The "Trust Store" section below goes into more
detail about how this is achieved.

Finally, this API will split the responsibilities currently assumed by the
`SSLContext`_ object: specifically, the responsibility for holding and managing
configuration and the responsibility for using that configuration to build
wrapper objects.

This is necessarily primarily for supporting functionality like Server Name
Indication (SNI). In OpenSSL (and thus in the ``ssl`` module), the server has
the ability to modify the TLS configuration in response to the client telling
the server what hostname it is trying to reach. This is mostly used to change
certificate chain so as to present the correct TLS certificate chain for the
given hostname. The specific mechanism by which this is done is by returning
a new `SSLContext`_ object with the appropriate configuration.

This is not a model that maps well to other TLS implementations. Instead, we
need to make it possible to provide a return value from the SNI callback that
can be used to indicate what configuration changes should be made. This means
providing an object that can hold TLS configuration. This object needs to be
applied to specific TLSWrappedBuffer, and TLSWrappedSocket objects.

For this reason, we split the responsibility of `SSLContext`_ into two separate
objects. The ``TLSConfiguration`` object is an object that acts as container
for TLS configuration: the ``ClientContext`` and ``ServerContext`` objects are
objects that are instantiated with a ``TLSConfiguration`` object. Both objects
would be immutable.

Configuration
~~~~~~~~~~~~~

The ``TLSConfiguration`` concrete class defines an object that can hold and
manage TLS configuration. The goals of this class are as follows:

1. To provide a method of specifying TLS configuration that avoids the risk of
   errors in typing (this excludes the use of a simple dictionary).
2. To provide an object that can be safely compared to other configuration
   objects to detect changes in TLS configuration, for use with the SNI
   callback.

This class is not an ABC, primarily because it is not expected to have
implementation-specific behaviour. The responsibility for transforming a
``TLSConfiguration`` object into a useful set of configuration for a given TLS
implementation belongs to the Context objects discussed below.

This class has one other notable property: it is immutable. This is a desirable
trait for a few reasons. The most important one is that it allows these objects
to be used as dictionary keys, which is potentially extremely valuable for
certain TLS backends and their SNI configuration. On top of this, it frees
implementations from needing to worry about their configuration objects being
changed under their feet, which allows them to avoid needing to carefully
synchronize changes between their concrete data structures and the
configuration object.

This object is extendable: that is, future releases of Python may add
configuration fields to this object as they become useful. For
backwards-compatibility purposes, new fields are only appended to this object.
Existing fields will never be removed, renamed, or reordered.

The ``TLSConfiguration`` object would be defined by the following code::

    ServerNameCallback = Callable[[TLSBufferObject, Optional[str], TLSConfiguration], Any]


    _configuration_fields = [
        'validate_certificates',
        'certificate_chain',
        'ciphers',
        'inner_protocols',
        'lowest_supported_version',
        'highest_supported_version',
        'trust_store',
        'sni_callback',
    ]


    _DEFAULT_VALUE = object()


    class TLSConfiguration(namedtuple('TLSConfiguration', _configuration_fields)):
        """
        An immutable TLS Configuration object. This object has the following
        properties:

        :param validate_certificates bool: Whether to validate the TLS
            certificates. This switch operates at a very broad scope: either
            validation is enabled, in which case all forms of validation are
            performed including hostname validation if possible, or validation
            is disabled, in which case no validation is performed.

            Not all backends support having their certificate validation
            disabled. If a backend does not support having their certificate
            validation disabled, attempting to set this property to ``False``
            will throw a ``TLSError`` when this object is passed into a
            context object.

        :param certificate_chain Tuple[Tuple[Certificate],PrivateKey]: The
            certificate, intermediate certificate, and the corresponding
            private key for the leaf certificate. These certificates will be
            offered to the remote peer during the handshake if required.

            The first Certificate in the list must be the leaf certificate. All
            subsequent certificates will be offered as intermediate additional
            certificates.

        :param ciphers Tuple[CipherSuite]:
            The available ciphers for TLS connections created with this
            configuration, in priority order.

        :param inner_protocols Tuple[Union[NextProtocol, bytes]]:
            Protocols that connections created with this configuration should
            advertise as supported during the TLS handshake. These may be
            advertised using either or both of ALPN or NPN. This list of
            protocols should be ordered by preference.

        :param lowest_supported_version TLSVersion:
            The minimum version of TLS that should be allowed on TLS
            connections using this configuration.

        :param highest_supported_version TLSVersion:
            The maximum version of TLS that should be allowed on TLS
            connections using this configuration.

        :param trust_store TrustStore:
            The trust store that connections using this configuration will use
            to validate certificates.

        :param sni_callback Optional[ServerNameCallback]:
            A callback function that will be called after the TLS Client Hello
            handshake message has been received by the TLS server when the TLS
            client specifies a server name indication.

            Only one callback can be set per ``TLSConfiguration``. If the
            ``sni_callback`` is ``None`` then the callback is disabled. If the
            ``TLSConfiguration`` is used for a ``ClientContext`` then this
            setting will be ignored.

            The ``callback`` function will be called with three arguments: the
            first will be the ``TLSBufferObject`` for the connection; the
            second will be a string that represents the server name that the
            client is intending to communicate (or ``None`` if the TLS Client
            Hello does not contain a server name); and the third argument will
            be the original ``Context``. The server name argument will be the
            IDNA *decoded* server name.

            The ``callback`` must return a ``TLSConfiguration`` to allow
            negotiation to continue. Other return values signal errors.
            Attempting to control what error is signaled by the underlying TLS
            implementation is not specified in this API, but is up to the
            concrete implementation to handle.

            The Context will do its best to apply the ``TLSConfiguration``
            changes from its original configuration to the incoming connection.
            This will usually include changing the certificate chain, but may
            also include changes to allowable ciphers or any other
            configuration settings.
        """
        __slots__ = ()

        def __new__(cls, validate_certificates=None: Optional[bool],
                         certificate_chain=None: Optional[Tuple[Tuple[Certificate], PrivateKey]],
                         ciphers=None: Optional[Tuple[CipherSuite]],
                         inner_protocols=None: Optional[Tuple[Union[NextProtocol, bytes]]],
                         lowest_supported_version=None: Optional[TLSVersion],
                         highest_supported_version=None: Optional[TLSVersion],
                         trust_store=None: Optional[TrustStore],
                         sni_callback=None: Optional[ServerNameCallback]):

            if validate_certificates is None:
                validate_certificates = True

            if ciphers is None:
                ciphers = DEFAULT_CIPHER_LIST

            if inner_protocols is None:
                inner_protocols = []

            if lowest_supported_version is None:
                lowest_supported_version = TLSVersion.TLSv1

            if highest_supported_version is None:
                highest_supported_version = TLSVersion.MAXIMUM_SUPPORTED

            return super().__new__(
                cls, validate_certificates, certificate_chain, ciphers,
                inner_protocols, lowest_supported_version,
                highest_supported_version, trust_store, sni_callback
            )

        def update(self, validate_certificates=_DEFAULT_VALUE,
                         certificate_chain=_DEFAULT_VALUE,
                         ciphers=_DEFAULT_VALUE,
                         inner_protocols=_DEFAULT_VALUE,
                         lowest_supported_version=_DEFAULT_VALUE,
                         highest_supported_version=_DEFAULT_VALUE,
                         trust_store=_DEFAULT_VALUE,
                         sni_callback=_DEFAULT_VALUE):
            """
            Create a new ``TLSConfiguration``, overriding some of the settings
            on the original configuration with the new settings.
            """
            if validate_certificates is _DEFAULT_VALUE:
                validate_certificates = self.validate_certificates

            if certificate_chain is _DEFAULT_VALUE:
                certificate_chain = self.certificate_chain

            if ciphers is _DEFAULT_VALUE:
                ciphers = self.ciphers

            if inner_protocols is _DEFAULT_VALUE:
                inner_protocols = self.inner_protocols

            if lowest_supported_version is _DEFAULT_VALUE:
                lowest_supported_version = self.lowest_supported_version

            if highest_supported_version is _DEFAULT_VALUE:
                highest_supported_version = self.highest_supported_version

            if trust_store is _DEFAULT_VALUE:
                trust_store = self.trust_store

            if sni_callback is _DEFAULT_VALUE:
                sni_callback = self.sni_callback

            return self.__class__(
                validate_certificates, certificate_chain, ciphers,
                inner_protocols, lowest_supported_version,
                highest_supported_version, trust_store, sni_callback
            )



Context
~~~~~~~

We define two Context abstract base classes. These ABCs define objects that
allow configuration of TLS to be applied to specific connections. They can be
thought of as factories for ``TLSWrappedSocket`` and ``TLSWrappedBuffer``
objects.

Unlike the current ``ssl`` module, we provide two context classes instead of
one. Specifically, we provide the ``ClientContext`` and ``ServerContext``
classes. This simplifies the APIs (for example, there is no sense in the server
providing the ``server_hostname`` parameter to ``ssl.SSLContext.wrap_socket``,
but because there is only one context class that parameter is still available),
and ensures that implementations know as early as possible which side of a TLS
connection they will serve. Additionally, it allows implementations to opt-out
of one or either side of the connection. For example, SChannel on macOS is not
really intended for server use and has an enormous amount of functionality
missing for server-side use. This would allow SChannel implementations to
simply not define a concrete subclass of ``ServerContext`` to signal their lack
of support.

As much as possible implementers should aim to make these classes immutable:
that is, they should prefer not to allow users to mutate their internal state
directly, instead preferring to create new contexts from new TLSConfiguration
objects. Obviously, the ABCs cannot enforce this constraint, and so they do not
attempt to.

The ``Context`` abstract base class has the following class definition::

    TLSBufferObject = Union[TLSWrappedSocket, TLSWrappedBuffer]


    class _BaseContext(metaclass=ABCMeta):
        @abstractmethod
        def __init__(self, configuration: TLSConfiguration):
            """
            Create a new context object from a given TLS configuration.
            """

        @property
        @abstractmethod
        def configuration(self) -> TLSConfiguration:
            """
            Returns the TLS configuration that was used to create the context.
            """


    class ClientContext(_BaseContext):
        @abstractmethod
        def wrap_socket(self,
                        socket: socket.socket,
                        server_hostname: Optional[str],
                        auto_handshake=True: bool) -> TLSWrappedSocket:
            """
            Wrap an existing Python socket object ``socket`` and return a
            ``TLSWrappedSocket`` object. ``socket`` must be a ``SOCK_STREAM``
            socket: all other socket types are unsupported.

            The returned SSL socket is tied to the context, its settings and
            certificates.

            The parameter ``server_hostname`` specifies the hostname of the
            service which we are connecting to. This allows a single server to
            host multiple SSL-based services with distinct certificates, quite
            similarly to HTTP virtual hosts. This is also used to validate the
            TLS certificate for the given hostname. If hostname validation is
            not desired, then pass ``None`` for this parameter.

            The parameter ``auto_handshake`` specifies whether to do the SSL
            handshake automatically after doing a ``socket.connect()``, or
            whether the application program will call it explicitly, by
            invoking the ``TLSWrappedSocket.do_handshake()`` method. Calling
            ``TLSWrappedSocket.do_handshake()`` explicitly gives the program
            control over the blocking behavior of the socket I/O involved in
            the handshake.
            """

        @abstractmethod
        def wrap_buffers(self, incoming: Any, outgoing: Any,
                         server_hostname: Optional[str]) -> TLSWrappedBuffer:
            """
            Wrap a pair of buffer objects (``incoming`` and ``outgoing``) to
            create an in-memory stream for TLS. The SSL routines will read data
            from ``incoming`` and decrypt it, and write encrypted data to
            ``outgoing``.

            The buffer objects must be either file objects or objects that
            implement the buffer protocol.

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
            handshake automatically after doing a ``socket.accept()``, or
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

            The buffer objects must be either file objects or objects that
            implement the buffer protocol.
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
        def cipher(self) -> Optional[CipherSuite]:
            """
            Returns the CipherSuite entry for the cipher that has been
            negotiated on the connection. If no connection has been negotiated,
            returns ``None``.
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
        def readinto(self, buffer: Any, amt=None: int) -> int:
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
            ``readinto()`` can also cause write operations.
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
        def cipher(self) -> Optional[CipherSuite]:
            """
            Returns the CipherSuite entry for the cipher that has been
            negotiated on the connection. If no connection has been negotiated,
            returns ``None``.
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

Supporting cipher suites in a truly library-agnostic fashion is a remarkably
difficult undertaking. Different TLS implementations often have *radically*
different APIs for specifying cipher suites, but more problematically these
APIs frequently differ in capability as well as in style. Some examples are
shown below:

OpenSSL
^^^^^^^

OpenSSL uses a well-known cipher string format. This format has been adopted as
a configuration language by most products that use OpenSSL, including Python.
This format is relatively easy to read, but has a number of downsides: it is
a string, which makes it remarkably easy to provide bad inputs; it lacks much
detailed validation, meaning that it is possible to configure OpenSSL in a way
that doesn't allow it to negotiate any cipher at all; and it allows specifying
cipher suites in a number of different ways that make it tricky to parse. The
biggest problem with this format is that there is no formal specification for
it, meaning that the only way to parse a given string the way OpenSSL would is
to get OpenSSL to parse it.

OpenSSL's cipher strings can look like this::

    'ECDH+AESGCM:ECDH+CHACHA20:DH+AESGCM:DH+CHACHA20:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!eNULL:!MD5'

This string demonstrates some of the complexity of the OpenSSL format. For
example, it is possible for one entry to specify multiple cipher suites: the
entry ``ECDH+AESGCM`` means "all ciphers suites that include both
elliptic-curve Diffie-Hellman key exchange and AES in Galois Counter Mode".
More explicitly, that will expand to four cipher suites::

    "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"

That makes parsing a complete OpenSSL cipher string extremely tricky. Add to
the fact that there are other meta-characters, such as "!" (exclude all cipher
suites that match this criterion, even if they would otherwise be included:
"!MD5" means that no cipher suites using the MD5 hash algorithm should be
included), "-" (exclude matching ciphers if they were already included, but
allow them to be re-added later if they get included again), and "+" (include
the matching ciphers, but place them at the end of the list), and you get an
*extremely* complex format to parse. On top of this complexity it should be
noted that the actual result depends on the OpenSSL version, as an OpenSSL
cipher string is valid so long as it contains at least one cipher that OpenSSL
recognises.

OpenSSL also uses different names for its ciphers than the names used in the
relevant specifications. See the manual page for ``ciphers(1)`` for more
details.

The actual API inside OpenSSL for the cipher string is simple::

    char *cipher_list = <some cipher list>;
    int rc = SSL_CTX_set_cipher_list(context, cipher_list);

This means that any format that is used by this module must be able to be
converted to an OpenSSL cipher string for use with OpenSSL.

SecureTransport
^^^^^^^^^^^^^^^

SecureTransport is the macOS system TLS library. This library is substantially
more restricted than OpenSSL in many ways, as it has a much more restricted
class of users. One of these substantial restrictions is in controlling
supported cipher suites.

Ciphers in SecureTransport are represented by a C ``enum``. This enum has one
entry per cipher suite, with no aggregate entries, meaning that it is not
possible to reproduce the meaning of an OpenSSL cipher string like
"ECDH+AESGCM" without hand-coding which categories each enum member falls into.

However, the names of most of the enum members are in line with the formal
names of the cipher suites: that is, the cipher suite that OpenSSL calls
"ECDHE-ECDSA-AES256-GCM-SHA384" is called
"TLS_ECDHE_ECDHSA_WITH_AES_256_GCM_SHA384" in SecureTransport.

The API for configuring cipher suites inside SecureTransport is simple::

    SSLCipherSuite ciphers[] = {TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384, ...};
    OSStatus status = SSLSetEnabledCiphers(context, ciphers, sizeof(cphers));

SChannel
^^^^^^^^

SChannel is the Windows system TLS library.

SChannel has extremely restrictive support for controlling available TLS
cipher suites, and additionally adopts a third method of expressing what TLS
cipher suites are supported.

Specifically, SChannel defines a set of ``ALG_ID`` constants (C unsigned ints).
Each of these constants does not refer to an entire cipher suite, but instead
an individual algorithm. Some examples are ``CALG_3DES`` and ``CALG_AES_256``,
which refer to the bulk encryption algorithm used in a cipher suite,
``CALG_DH_EPHEM`` and ``CALG_RSA_KEYX`` which refer to part of the key exchange
algorithm used in a cipher suite, ``CALG_SHA1`` and ``CALG_MD5`` which refer to
the message authentication code used in a cipher suite, and ``CALG_ECDSA`` and
``CALG_RSA_SIGN`` which refer to the signing portions of the key exchange
algorithm.

This can be thought of as the half of OpenSSL's functionality that
SecureTransport doesn't have: SecureTransport only allows specifying exact
cipher suites, while SChannel only allows specifying *parts* of the cipher
suite, while OpenSSL allows both.

Determining which cipher suites are allowed on a given connection is done by
providing a pointer to an array of these ``ALG_ID`` constants. This means that
any suitable API must allow the Python code to determine which ``ALG_ID``
constants must be provided.


Proposed Interface
^^^^^^^^^^^^^^^^^^

The proposed interface for the new module is influenced by the combined set of
limitations of the above implementations. Specifically, as every implementation
*except* OpenSSL requires that each individual cipher be provided, there is no
option but to provide that lowest-common denominator approach.

The simplest approach is to provide an enumerated type that includes all of the
cipher suites defined for TLS. The values of the enum members will be their
two-octet cipher identifier as used in the TLS handshake, stored as a tuple of
integers. The names of the enum members will be their IANA-registered cipher
suite names.

Rather than populate this enum by hand, it is likely that we'll define a
script that can build it from Christian Heimes' `tlsdb JSON file`_ (warning:
large file). This also opens up the possibility of extending the API with
additional querying function, such as determining which TLS versions support
which ciphers, if that functionality is found to be useful or necessary.

If users find this approach to be onerous, a future extension to this API can
provide helpers that can reintroduce OpenSSL's aggregation functionality.

Because this enum would be enormous, the entire enum is not provided here.
Instead, a small sample of entries is provided to give a flavor of how it will
appear.

::

    class CipherSuite(Enum):
        ...
        TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA = (0xC0, 0x12)
        ...
        TLS_ECDHE_ECDSA_WITH_AES_128_CCM = (0xC0, 0xAC)
        ...
        TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 = (0xC0, 0x2B)
        ...


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
many of the the other classes defined here, these classes are not abstract, as
they have no behaviour. They exist simply to signal certain common behaviours.
Backends should subclass these exceptions in their own packages, but needn't
define any behaviour for them.

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

Concrete implementations should aim to have Certificate objects be hashable if
at all possible. This will help ensure that TLSConfiguration objects used with
an individual concrete implementation are also hashable.

::

    class Certificate(metaclass=ABCMeta):
        @abstractclassmethod
        def from_buffer(cls, buffer: bytes) -> Certificate:
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
        def from_file(cls, path: Union[pathlib.Path, AnyStr]) -> Certificate:
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

::

    class PrivateKey(metaclass=ABCMeta):
        @abstractclassmethod
        def from_buffer(cls,
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
        def from_file(cls,
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

Concrete implementations should aim to have TrustStore objects be hashable if
at all possible. This will help ensure that TLSConfiguration objects used with
an individual concrete implementation are also hashable.

::

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


Runtime Access
~~~~~~~~~~~~~~

A not-uncommon use case for library users is to want to allow the library to
control the TLS configuration, but to want to select what backend is in use.
For example, users of Requests may want to be able to select between OpenSSL or
a platform-native solution on Windows and macOS, or between OpenSSL and NSS on
some Linux platforms. These users, however, may not care about exactly how
their TLS configuration is done.

This poses a problem: given an arbitrary concrete implementation, how can a
library work out how to load certificates into the trust store? There are two
options: either all concrete implementations can be required to fit into a
specific naming scheme, or we can provide an API that makes it possible to grab
these objects.

This PEP proposes that we use the second approach. This grants the greatest
freedom to concrete implementations to structure their code as they see fit,
requiring only that they provide a single object that has the appropriate
properties in place. Users can then pass this "backend" object to libraries
that support it, and those libraries can take care of configuring and using the
concrete implementation.

All concrete implementations must provide a method of obtaining a ``Backend``
object. The ``Backend`` object can be a global singleton or can be created by a
callable if there is an advantage in doing that.

The ``Backend`` object has the following definition::

    Backend = namedtuple(
        'Backend',
        ['client_context', 'server_context',
         'certificate', 'private_key', 'trust_store']
    )

Each of the properties must provide the concrete implementation of the relevant
ABC. This ensures that code like this will work for any backend::

    trust_store = backend.trust_store.system()


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
.. _MSDN articles: https://msdn.microsoft.com/en-us/library/windows/desktop/mt490158(v=vs.85).aspx
.. _tlsdb JSON file: https://github.com/tiran/tlsdb/blob/master/tlsdb.json


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
