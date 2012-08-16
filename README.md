## Samlr [![Build Status](https://secure.travis-ci.org/morten/samlr.png)](http://travis-ci.org/morten/samlr)

Samlr is a clean room implementation of SAML for Ruby. It's focused on implementing the service provider (SP) side rather than the identity provider (IdP).

Samlr leverages Nokogiri for the heavy lifting and keeps things simple. Samlr allows you to receive and validate SAML authentication requests. It's SAML 2.0 only, doesn't support everything and makes liberal assumptions about the input - none of which cannot be improved going forward.

### Initiating an authentication request

```ruby
saml_request = Samlr::Request.new(
  :issuer               => request.host,
  :name_identity_format => Samlr::EMAIL_FORMAT,
  :consumer_service_url => "https://#{request.host}/auth/saml"
)
```

At this point you can access `request.param` if all you want is the encoded params, or you can get a fully valid request URL with an appropriate `RelayState` value:

```ruby
redirect_to saml_request.url(
  "https://idp.example.com/auth/saml", { :RelayState => request.url }
)
```

Once the IdP receives the request, it prompts the user to authenticate, after which it sends the SAML response to your application.

### Verifying a SAML response

You can validate a SAML response string using either of the below approaches. The fingerprint is a certificate fingerprint, and the certificate is the certificate PEM (from which Samlr will obtain the fingerprint).

```ruby
saml_response = Samlr::Response.new(params[:SAMLResponse], :fingerprint => fingerprint)
```

Or using a certificate:

```ruby
saml_response = Samlr::Response.new(params[:SAMLResponse], :certificate => certificate)
```

You then verify the response by calling

```ruby
saml_response.verify!
```

If the verification fails for whatever reason, a `Samlr::Error` will be thrown. This error class has several subclasses and generally contains a useful error message that can help trouble shooting. The error also has a `Samlr::Error#details` value, which contains potentially sensitive data (fingerprint values, canonicalization results).

```ruby
begin
  saml_response.verify!
  redirect_to success!(saml_response.name_id)
rescue Samlr::SamlrError => e
  logger.warn("SAML error #{e.class} #{e.message} #{e.details}")
  flash[:error] = e.message
end
```

When the verification suceeds,the resulting response object will surface `saml_response.name_id` (String) and `saml_response.attributes` (Hash).

### Metadata

Currently no support for signing, but that should be fairly easy to extract from the `Samlr::Tools::ResponseBuilder`. Get a metadata XML document like this:

```ruby
xml = Samlr::Tools::MetadataBuilder.build({
  :entity_id            => "https://sp.example.com/saml",
  :name_identity_format => Samlr::EMAIL_FORMAT,
  :consumer_service_url => "https://sp.example.com/saml"
})
```

### Command line

Useful to work with files, e.g.

```
$ samlr -v --skip-conditions -f 83:CC:12:...:F7:9D:19 response.xml.base64
$ Verification passed
```

Run `samlr -h` for options.

```
SAML response command line tool.

Usage examples:
  samlr --verify --fingerprint ab:23:cd --skip-conditions response.xml
  samlr --verify --skip-fingerprint --skip-conditions response.xml
  samlr --schema-validate response.xml
  samlr --print response.xml.base64

Full list of options:
            --verify, -v:   Verify a SAML response document
   --fingerprint, -f <s>:   The fingerprint to verify the certificate against
   --skip-conditions, -s:   Skip conditions check
  --skip-fingerprint, -k:   Skip certificate fingerprint check
   --schema-validate, -c:   Perform a schema validation against the input
             --print, -p:   Pretty prints the XML
              --help, -h:   Show this message
```

### Testing

```
bundle install
rake
```

### Supported IdPs

Please help adding IdP's or IdP services you find to work with Samlr. The below list of are known to work:

* Novell/NetIQ
* MS ADFS 2.0
* http://simplesamlphp.org/
* http://www.ssoeasy.com/
* http://www.okta.com/
* http://www.onelogin.com/
* [Salesforce SAML IdP](https://login.salesforce.com/help/doc/en/identity_provider_about.htm)

### Security

As part of keeping things secure, Samlr does schema validation on all response documents. You can control what it should do in case of invalid documents:

1. Reject the request. This is the recommended and default.
2. Log the errorneous document. In case you're transitioning to Samlr and want reassurance.

You control this by setting the schema validation mode in e.g. an initializer

```ruby
Samlr.validation_mode = :reject
Samlr.validation_mode = :log
```

### Logging

Samlr has a (silent) default logger that prints to STDOUT. You can change the log level of this logger if you want to see the output:

```ruby
Samlr.logger.level = Logger::DEBUG
```

Or you can replace the logger altogether

```ruby
Samlr.logger = Rails.logger
```

### Contributing

Pull requests very welcome. Write tests. Adhere to standards employed (indentation, spaces vs. tabs etc.).

### Error reporting

Pull requests with a failing test case much preferred.
