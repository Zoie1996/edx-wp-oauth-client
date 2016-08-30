# edx_wp_oauth_client
SSO Client for [Wordpress OAuth plugin provider][wp_oauth_provider].
### Instalation guide
 - Install  WP plugin following instruction. In wp-admin OAuth Server tab add new client.
Redirect uri must be **http://edx_url/auth/complete/wp-oauth2/**

 - Install this client
   ```
   pip install -e git+https://github.com/xahgmah/edx-wp-oauth-client.git#egg=edx_wp_oauth_client
   ```

 - Enable THIRD_PARTY_AUTH in edX
 
    In the edx/app/edxapp/lms.env.json file, edit the file so that it includes the following line in the features section.       And add  this backend.
    ```
    ...
    "FEATURES" : {
        ...
        "ENABLE_COMBINED_LOGIN_REGISTRATION": true,
        "ENABLE_THIRD_PARTY_AUTH": true,
        "WP_PROVIDER_URL": "<URL OF YOUR SSO>"
    }
    ...
    "THIRD_PARTY_AUTH_BACKENDS":["edx_wp_oauth_client.backends.wp_oauth_client.WPOAuthBackend"]
    ```
   
 - Add in file **lms/envs/common.py**. It's preffered to place it somewhere at the top of the list
    ```
    INSTALLED_APPS = (
        ...
        'edx_wp_oauth_client',
        ...
    )
    ```
    
 - Add provider config in edX admin panel /admin/third_party_auth/oauth2providerconfig/
   - Enabled - **true**
   - backend-name - **wp-oauth2**
   - Skip registration form - **true**
   - Skip email verification - **true**
   - Client ID from WP Admin OAuth Tab
   - Client Secret from WP Admin OAuth Tab
    
 - If you're want seamless authorization add middleware classes for SeamlessAuthorization (crossdomain cookie support needed)
   ```
   MIDDLEWARE_CLASSES += ("edx_wp_oauth_client.middleware.SeamlessAuthorization",)
   ```
   
   And add this code in the end of **functions.php** for your Wordpress theme
   ```
    $auth_cookie_name = "authenticated";
    $domain_name = "<YOUR_DOMAIN>";
    
    add_action('wp_login', 'set_auth_cookie', 1, 2);
    function set_auth_cookie($user_login, $user)
    {
        /**
         * After login set multidomain cookies which gives to edx understanding that user have already registrated
         */
        global $auth_cookie_name, $domain_name;
        setcookie($auth_cookie_name, 1, time() + 60 * 60 * 24 * 30, "/", ".{$domain_name}");
        setcookie($auth_cookie_name . "_user", $user->nickname, time() + 60 * 60 * 24 * 30, "/", ".{$domain_name}");
    }
    
    add_action('wp_logout', 'remove_custom_cookie_admin');
    function remove_custom_cookie_admin()
    {
        /**
         * After logout delete multidomain cookies which was added above
         */
        global $auth_cookie_name, $domain_name;
        setcookie($auth_cookie_name, "", time() - 3600, "/", ".{$domain_name}");
        setcookie($auth_cookie_name . "_user", "", time() - 3600, "/", ".{$domain_name}");
    }
    
    add_action('user_register', 'create_edx_user_after_registration', 10, 1);
    
    function create_edx_user_after_registration($user_id)
    {
        /**
         * Create edX user after user creation on Wordpress. This hack allows make API requests to edX before
         * the user visit edX first time.
         * Also this function allows update user data by wordpress initiative
         */
        global $wpdb, $domain_name;
        # fix this url with your LMS address
        $client_url = "https://courses.{$domain_name}/auth/complete/wp-oauth2/";
        $query = "SELECT * FROM `wp_oauth_clients` WHERE `redirect_uri` = '{$client_url}'";
        $client = $wpdb->get_row($query);
        if ($client) {
            require_once ABSPATH . '/wp-content/plugins/oauth2-provider/library/OAuth2/Autoloader.php';
            OAuth2\Autoloader::register();
            $storage = new OAuth2\Storage\Wordpressdb();
            $authCode = new OAuth2\OpenID\ResponseType\AuthorizationCode($storage);
            $code = $authCode->createAuthorizationCode($client->client_id, $user_id, $client->redirect_uri);
            $params = http_build_query(array(
                'state' => md5(time()),
                'code' => $code
            ));
            file_get_contents($client->redirect_uri . "?" . $params);
        }
    }
   ```

 
**Note.** If you work on local devstack. Inside your edx’s vagrant in /etc/hosts add a row with your machines’s IP  and wordpress’s >vhost. For example:
```192.168.0.197 wp.local```

[wp_oauth_provider]: <https://wordpress.org/plugins/oauth2-provider/>
