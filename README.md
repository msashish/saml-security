## Initial setup https://www.baeldung.com/spring-security-saml

    1) Create project from https://start.spring.io/ and adding web, security as dependency
    Change pom to add opensaml and spring-security-saml2-core dependencies
    
    2) Setup Okta account following https://developer.okta.com/ and create new web app integration with SAML 2.0
            SSO URL: http://localhost:8080/saml/SSO
            Audience URI: http://localhost:8080/saml/metadata
            App username: Okta username
        
       Once done, get following from setup instructions:
            Idp SSO URL
            Idp Issuer
       
       Important: Add user (yourself) to the application by using Assign user tab
    
    3) Create a self-signed key and Keystore
        keytool -genkeypair -alias springo -keypass changeitashish -keystore /Users/sheelava/msashishgit/saml-security/src/main/resources/saml/samlKeystore.jks -keyalg RSA -keysize 2048 -validity 10000
    
    4) Update application.properties with keystore, Idp details
    
## Springboot SAML configuration code

    1) Create config/SamlSecurityConfig
    
        SAMLEntryPoint class that will work as an entry point for SAML authentication:
        WebSSOProfileOptions bean allows us to set up parameters of the request sent from SP to IdP asking for user authentication
           
               public SAMLEntryPoint samlEntryPoint() 
               public WebSSOProfileOptions defaultWebSSOProfileOptions()
               
               SimpleUrlLogoutSuccessHandler successLogoutHandler()
               public SecurityContextLogoutHandler logoutHandler()
               public SAMLLogoutProcessingFilter samlLogoutProcessingFilter()
               public SAMLLogoutFilter samlLogoutFilter()
        
    2) Create config/WebSecurityConfig
        
        Create a few filters for our SAML URIs like /discovery, /login, and /logout
        
                public FilterChainProxy samlFilter() 
                public SAMLProcessingFilter samlWebSSOProcessingFilter()
                public SAMLDiscovery samlDiscovery() 
                public SavedRequestAwareAuthenticationSuccessHandler successRedirectHandler()
                public SimpleUrlAuthenticationFailureHandler authenticationFailureHandler()
        
    
    3) Create authentication/CustomSAMLAuthenticationProvider
    
    4) Metadata handling - we'll provide IdP metadata XML to the SP. 
        It'll help to let our IdP know which SP endpoint it should redirect to once the user is logged in.
        
        In WebSecurityConfig:
         public MetadataGenerator metadataGenerator()
         public MetadataGeneratorFilter metadataGeneratorFilter() 
         public ExtendedMetadata extendedMetadata()
         
        In SamlSecurityConfig: The MetadataGenerator bean requires an instance of the KeyManager to encrypt the 
        exchange between SP and IdP:
            public KeyManager keyManager()
            public ExtendedMetadataDelegate oktaExtendedMetadataProvider()  - we'll configure the IdP metadata into our Spring Boot application using the ExtendedMetadataDelegate instance:
            public CachingMetadataManager metadata()
            
         Since communication will be in XML, add XML parser and processor:
            public StaticBasicParserPool parserPool()
            public ParserPoolHolder parserPoolHolder()
            public HTTPPostBinding httpPostBinding()
            public HTTPRedirectDeflateBinding httpRedirectDeflateBinding()
            public SAMLProcessorImpl processor() 
            
    5) Write CustomSAMLAuthenticationProvider 
        we require a custom implementation of the SAMLAuthenticationProvider class to check the instance of the 
        ExpiringUsernameAuthenticationToken class and set the obtained authorities:
        
            public class CustomSAMLAuthenticationProvider extends SAMLAuthenticationProvider 
        We should configure the CustomSAMLAuthenticationProvider as a bean in the SecurityConfig class:   
                  
    6) SecurityConfig - configure a basic HTTP security using the already discussed samlEntryPoint and samlFilter:
        
        In WebSecurityConfig:
            protected void configure(HttpSecurity http)
             
## Workings of SAML security
        - When the user tries to log in for the first time, the samlEntryPoint will handle the entry request. 
        - Then, the samlDiscovery bean (if enabled) will discover the IdP to contact for authentication.    
        - User is presented login by IdP
        - Next, when the user logs in, the IdP redirects the SAML response to the /saml/sso URI (setup at IdP) for processing, 
            and corresponding samlWebSSOProcessingFilter will authenticate the associated auth token.
        - When successful, the successRedirectHandler will redirect the user to the default target URL (/home). 
        - Otherwise, the authenticationFailureHandler will redirect the user to the /error URL.
        
## Test
    mvn compile
    mvn spring-boot:run 
    Check at http://localhost:8080/
    
