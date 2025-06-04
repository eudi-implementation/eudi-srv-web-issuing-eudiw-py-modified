# Modified EUDIW Issuer

This Repository is a modified version of the [EUDIW Reference Issuer (Python)](https://github.com/eu-digital-identity-wallet/eudi-srv-web-issuing-eudiw-py).

## Setting up the Modified Issuer

[SETUP](SETUP.md) explains how to modify the original EUDIW Issuer to run locally.
In the present repository, the modifications in [4. How to add credentials?](SETUP.md#4-how-to-add-credentials) and [5. Connect to DC4EU datastore](SETUP.md#5-connect-to-dc4eu-datastore) are already implemented (plus a few extra ones).
They are included in the guide to explain how this repository was modified and therefore differs from the original.
For a detailed overview of the changes, compare the commits to the initial commit (which mirrors the original version).

The missing modifications are explained in [3. How to run the EUDIW Issuer?](SETUP.md#3-how-to-run-the-eudiw-issuer).
They are missing because virtual environments, certs, and keys should not be shared via git.

### Additional changes not reflected in the guide include: 

- Supported credentials include a modified PID, EHIC, and PDA1 credential. All other credentials were moved to `app/metadata_config/old_credentials_supported`.
- Additional line in `.gitignore` to ignore everything in `app/certs` so that keys and certs aren't involuntarily shared.
- Additional line in `.gitignore` to ignore `.venv`.
- Additional line in `.gitignore` to ignore `.idea`. If you're using a different IDE, make sure to not add its config files to `.gitignore` as well.

### Further possible changes

#### 1. Modify Logos

- You can modify the logo that will be displayed in the wallet by adding it to `app/static/images` and then reference it in `app/metadata_config/metadata_config.json`:
    ```json
    "display": [
      {
        "name": "Company Name",
        "locale": "en",
        "logo": {
          "uri": "https://your.local.ip.address:5000/logo.png",
          "alt_text": "Company Logo"
        }
      }
    ],
    ```
    Also, you have to modify `__init__.py` to serve this new route:    
    ```python
    @app.route("/logo.png")
    def logo():
        return send_from_directory("static/images", "logo.png")
    ```

    Be aware that the wallet performs an SSL-Handshake to retrieve the logo.
    If your server certificate is not signed by a trusted certificate authority, it will either fail and not display properly (which doesn't influence the performance of the wallet per se) or you can modify the wallet to trust user-installed certificates.
    If you want to do the latter, remember to include it in the modifications of the wallet and then install the `my-root-ca.crt` that you create during the setup of this issuer on your phone on which the wallet runs.

- You can also modify the logo that will be displayed in the browser. For this, replace `app/static/ic-logo.svg` with your logo.
    Pay attention that they logos are either roughly the same height or specify
    ```html
    <img src="{{url_for('static', filename='ic-logo.svg')}}" height="40">
    ``` 
  whenever the logo is used (take a look at the files in `app/templates`).