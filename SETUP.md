# Setup on Linux

## 1. Python

The EUDIW Issuer was tested with

+ Python version 3.9.2

and should only be used with Python 3.9 or 3.10.

If you don't have it installed, please download it from <https://www.python.org/downloads/> and follow
the [Python Developer's Guide](https://devguide.python.org/getting-started/).

## 2. Flask

The EUDIW Issuer was tested with

+ Flask v. 2.3

and should only be used with Flask v. 2.3 or higher.

Flask will be installed later via `requirements.txt`. To
install [Flask](https://flask.palletsprojects.com/en/2.3.x/) manually, please follow
the [Installation Guide](https://flask.palletsprojects.com/en/2.3.x/installation/).

## 3. How to run the EUDIW Issuer?

To run the EUDIW Issuer, please follow these simple steps (some of which may have already been completed when installing
Flask) for Linux/macOS or Windows.

### 1. Clone the EUDIW Issuer repository:

   If you're setting up everything yourself, clone the original repo:
   ```shell
   git clone git@github.com:eu-digital-identity-wallet/eudi-srv-web-issuing-eudiw-py.git
   ```

   If you rely on the modifications made here, clone this repo:
   ```shell
   git clone git@github.com:eudi-implementation/eudi-srv-web-issuing-eudiw-py-modified.git
   ```

### 2. Create a `.venv` folder within the cloned repository:

  ```shell
  cd eudi-srv-web-issuing-eudiw-py
  python3 -m venv .venv
  ```

### 3. Activate the environment:

  ```shell
  . .venv/bin/activate
  ```

### 4. Install or upgrade _pip_

  ```shell
  python -m pip install --upgrade pip
  ```

### 5. Install Flask, gunicorn and other dependencies in virtual environment

  ```shell
  pip install -r app/requirements.txt
  ```

### 6. Setup secrets

- Copy `app/app_config/__config_secrets.py` to `app/app_config/config_secrets.py` and modify secrets.

- Open a python shell:
  ```shell
  python
  ```

- Generate secrets:
  ```python
  import os
  os.urandom(16).hex()
  ```

- Exit python shell:

  ```python
  exit()
  ```

### 7. Setup IACA certs and keys

- Create directories:
  ```shell
  mkdir certs
  mkdir certs/IACA
  ```
- Extract `test_tokens/IACA-token/PIDIssuerCAUT01.pem.gz` to `certs/IACA`
- Extract `test_tokens/DS-token/PID-DS-0002.zip` to `certs`
- Decrypt the private key by running the following command
  `openssl ec -in certs/PID-DS-0002/PID-DS-0002.pid-ds-0002.key.pem -out certs/PID-DS-0002/PID-DS-0002-decrypted.key.pem`
  with password "pid-ds-0002".

### 8. Setup Certificates

- Create directories:
  ```shell
  mkdir certs/ca
  mkdir certs/your.local.ip.address
  ```

- Create own root CA:
  ```shell
  openssl req -x509 -new -nodes -keyout certs/ca/my-root-ca.key -out certs/ca/my-root-ca.crt -days 3650 -subj "/CN=MyRootCA"
  ```

- Generate server private key:
  ```shell
  openssl req -new -nodes \
  -out certs/your.local.ip.address/server.csr \
  -keyout certs/your.local.ip.address/server.key \
  -subj "/CN=your.local.ip.address"
  ```

- Sign CSR with root CA:
  ```shell
  openssl x509 -req \
  -in certs/your.local.ip.address/server.csr \
  -CA certs/ca/my-root-ca.crt \
  -CAkey certs/ca/my-root-ca.key \
  -CAcreateserial \
  -out certs/your.local.ip.address/server.crt \
  -days 825 \
  -extfile <(printf "subjectAltName=IP:your.local.ip.address")
  ```

- Copy root CA to system-wide trust store:
  ```shell
  sudo cp certs/ca/my-root-ca.crt /usr/local/share/ca-certificates/
  sudo update-ca-certificates
  ```

### 9. Service Configuration

Base configuration for the EUDIW Issuer is located in `app/app_config/config_service.py`.

Parameters that should be changed:

- `service_url` (Base url of the service) (= `"https://your.local.ip.address:5000"`)
- `trusted_CAs_path` (Path to a folder with trusted IACA certificates) (= `"./certs/IACA"`)

### 10. Configuration of Countries

The supported countries configuration of the EUDIW Issuer is located in `/app/app_config/config_countries.py`.

For dev purposes we adapt formCountry as follows:

  ```json
  "pid_mdoc_privkey": "./certs/PID-DS-0002/PID-DS-0002-decrypted.key.pem",
"pid_mdoc_privkey_passwd": None,
"pid_mdoc_cert": "./certs/PID-DS-0002/PID-DS-0002.cert.der",
  ```

We will use formCountry to issue credentials.

### 11. Metadata configuration

The EUDIW Issuer OAuth2 metadata configuration files are located in `app/metadata_config/metadata_config.json` and
`app/metadata_config/openid-configuration.json`

You must change the base URL of the endpoints from `https://issuer.eudiw.dev` to
`https://your.local.ip.address:5000`

(It might be necessary to remove the nonce endpoint in `metadata_config.json`)

### 12. Run the EUDIW Issuer

- Add root ca to environment variables:
  ```shell
  export REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
  ```

- Run the issuer with the following command (in venv):
  ```shell
  flask --app app run --cert=./certs/your.local.ip.address/server.crt --key=./certs/your.local.ip.address/server.key --host=0.0.0.0
  ```

DO NOT RUN IN DEBUG MODE. Debug mode with `--host=0.0.0.0` enables arbitrary code execution on your machine for all
devices in your network.

## 4. How to add credentials?

To add new credentials, we have to add the .json-schemes to `metadata_config/credentials_supported`. The issuer offers
all credentials defined in this folder. If we want to limit the amount of credentials the issuer offers, we have to
remove them from this folder.

### 1. Create New Credential Schemes

Create new .json-file `new_credential.json`, e.g., `ehic_drv_mdoc.json`. Make sure to rename the highest-level key and "
scope" accordingly (`"eu.europa.ec.eudi.ehic_drv_mdoc"`). If you re-define an existing credential, you can keep the
format, doctype, etc. and continue by modifying the claims. To easily distinguish the new credential from the old one,
it may be useful to change the `"display.name"` as well.

`ehic_drv_mdoc.json`:
  ```json
  {
    "eu.europa.ec.eudi.ehic_drv_mdoc": {
      "format": "mso_mdoc",
      "doctype": "eu.europa.ec.eudi.ehic.1",
      "scope": "eu.europa.ec.eudi.ehic_drv_mdoc",
      "policy": {
        "batch_size": 50,
        "one_time_use": true
      },
      "cryptographic_binding_methods_supported": [
        "jwk",
        "cose_key"
      ],
      "credential_alg_values_supported": [
        -7
      ],
      "credential_crv_values_supported": [
        1
      ],
      "credential_signing_alg_values_supported": [
        "ES256"
      ],
      "proof_types_supported": {
        "jwt": {
          "proof_signing_alg_values_supported": [
            "ES256"
          ]
        }
      },
      "display": [
        {
          "name": "EHIC (DRV)",
          "locale": "en",
          "logo": {
            "uri": "https://examplestate.com/public/pid.png",
            "alt_text": "A square figure of a PID"
          }
        }
      ],
      "claims": [
        {
          "path": [
            "eu.europa.ec.eudi.ehic.1",
            "subject"
          ],
          "mandatory": true,
          "value_type": "subject_attributes",
          "source": "user",
          "display": [
            {
              "name": "Subject",
              "locale": "en"
            }
          ],
          "issuer_conditions": {
            "cardinality": {
              "min": 0,
              "max": 1
            },
            "subject_attributes": {
              "family_name": {
                "mandatory": true,
                "value_type": "string",
                "source": "user"
              },
              "given_name": {
                "mandatory": true,
                "value_type": "string",
                "source": "user"
              },
              "birth_date": {
                "mandatory": true,
                "value_type": "full-date",
                "source": "user"
              }
            }
          }
        },
        {
          "path": [
            "eu.europa.ec.eudi.ehic.1",
            "social_security_pin"
          ],
          "mandatory": true,
          "value_type": "string",
          "source": "user",
          "display": [
            {
              "name": "Social Security Pin",
              "locale": "en"
            }
          ]
        },
        {
          "path": [
            "eu.europa.ec.eudi.ehic.1",
            "starting_date"
          ],
          "mandatory": true,
          "value_type": "full-date",
          "source": "user",
          "display": [
            {
              "name": "Starting Date",
              "locale": "en"
            }
          ]
        },
        {
          "path": [
            "eu.europa.ec.eudi.ehic.1",
            "ending_date"
          ],
          "mandatory": true,
          "value_type": "full-date",
          "source": "user",
          "display": [
            {
              "name": "Ending Date",
              "locale": "en"
            }
          ]
        },
        {
          "path": [
            "eu.europa.ec.eudi.ehic.1",
            "document_id"
          ],
          "mandatory": true,
          "value_type": "string",
          "source": "user",
          "display": [
            {
              "name": "Document ID",
              "locale": "en"
            }
          ]
        },
        {
          "path": [
            "eu.europa.ec.eudi.ehic.1",
            "competent_institution"
          ],
          "mandatory": true,
          "value_type": "competent_institution_attributes",
          "source": "user",
          "display": [
            {
              "name": "Institution ID",
              "locale": "en"
            }
          ],
          "issuer_conditions": {
            "cardinality": {
              "min": 1,
              "max": 1
            },
            "competent_institution_attributes": {
              "institution_id": {
                "mandatory": true,
                "value_type": "string",
                "source": "user"
              },
              "institution_name": {
                "mandatory": false,
                "value_type": "string",
                "source": "user"
              },
              "country_code": {
                "mandatory": true,
                "value_type": "string",
                "source": "user"
              }
            }
          }
        },
        {
          "path": [
            "eu.europa.ec.eudi.ehic.1",
            "issuance_date"
          ],
          "mandatory": true,
          "source": "issuer",
          "display": [
            {
              "name": "Issuance Date",
              "locale": "en"
            }
          ]
        },
        {
          "path": [
            "eu.europa.ec.eudi.ehic.1",
            "expiry_date"
          ],
          "mandatory": true,
          "source": "issuer",
          "display": [
            {
              "name": "Expiry Date",
              "locale": "en"
            }
          ]
        },
        {
          "path": [
            "eu.europa.ec.eudi.ehic.1",
            "issuing_authority"
          ],
          "mandatory": true,
          "source": "issuer",
          "display": [
            {
              "name": "Issuance Authority",
              "locale": "en"
            }
          ]
        },
        {
          "path": [
            "eu.europa.ec.eudi.ehic.1",
            "issuing_country"
          ],
          "mandatory": true,
          "source": "issuer",
          "display": [
            {
              "name": "Issuing Country",
              "locale": "en"
            }
          ]
        }
      ],
      "issuer_config": {
        "issuing_authority": "Test PID issuer",
        "organization_id": "EUDI Wallet Reference Implementation",
        "validity": 90,
        "organization_name": "Test PID issuer",
        "namespace": "eu.europa.ec.eudi.ehic.1"
      }
    }
  }
  ```

### 2. Add Credential to formCountry

In `app_config/config_countries.py` add `"eu.europa.ec.eudi.ehic_drv_mdoc"` to `"formCountry.supported_credentials"`.

### 3. Define Authentication Method for New Credential

In `app_config/config_service.py` we can specify the authentication method for our new credential by adding it to `"auth_method_supported_credentials.PID_login"` and/or `"auth_method_supported_credentials.country_selection"`.
If we want to use PID authentication, we have to specify which attributes need to be requested. Add the following to `"dynamic_issuing"`:
```python
"eu.europa.ec.eudi.ehic_drv_mdoc": {
  "eu.europa.ec.eudi.pid.1": {
    "eu.europa.ec.eudi.pid.1": [
      "family_name",
      "given_name",
      "birth_date",
      "authentic_source_person_id",
      "issuing_country",
      "nationality",
    ]
  }
},
```

## 5. Connect to DC4EU datastore

We assume the DC4EU docker containers (version 0.5.22) are running on the same host and the apigw is exposed on port 9080. You can find the repository with these changes [here](https://github.com/eudi-implementation/dc4eu-vc-minmod).

### 1. Add Collect ID to Credential Offer

#### Frontend
In `templates/openid/credential_offer.html`, change the credential type selection from checkbox to radio. This way only one credential can be offered at a time.
```html
<label class="form-check-label" for="flexCheckDefault">
  <input class="form-check-input" id="check" type="radio" name="credential_id" value="{{ credential_id }}"
         style="transform: scale(1.8);">
  {{ cred[credential_type][credential_id]}}
</label>
```
Furthermore, before ```<div class="clearfix"></div>``` add an input field for the collect id:
```html

<div id="issuerStateContainer" class="form-group" style="display: none; margin-top: 2%;">
  <label for="issuer_state">Collect ID (required for EHIC (DRV) or PDA1 (DRV)):</label>
  <input type="text" class="form-control" name="issuer_state" id="issuer_state" placeholder="Enter Collect ID">
</div>
```
Lastly, replace the existing script at the bottom with:
```html
<script>

  $(document).ready(function () {
    function checkIssuerStateRequirement() {
      let selected = $("input[type='radio'][name='credential_id']:checked").val();

      const needsIssuerState = selected === "eu.europa.ec.eudi.ehic_drv_mdoc" || selected === "eu.europa.ec.eudi.pda1_drv_mdoc";

      if (needsIssuerState) {
        $("#issuerStateContainer").show();
        $("#issuer_state").attr("required", true);
      } else {
        $("#issuerStateContainer").hide();
        $("#issuer_state").val("");
        $("#issuer_state").removeAttr("required");
      }
    }

    $("input[type='radio'][name='credential_id']").change(function () {
      $('#btncheck').prop("disabled", false);
      checkIssuerStateRequirement();
    });

    // Run once on page load in case form is pre-filled (e.g. back/forward nav)
    checkIssuerStateRequirement();
  });

</script>
```

#### Backend
In `route_oidc.py`, replace
```python
if "proceed" in form_keys:
  form = list(form_keys)
  form.remove("proceed")
  form.remove("credential_offer_URI")
  form.remove("Authorization Code Grant")
  all_exist = all(credential in credentialsSupported for credential in form)

  if all_exist:
    credentials_id = form
    session["credentials_id"] = credentials_id
    credentials_id_list = json.dumps(form)
    if auth_choice == "pre_auth_code":
      session["credential_offer_URI"] = credential_offer_URI
      return redirect(
        url_for("preauth.preauthRed", credentials_id=credentials_id_list)
      )

    else:

      credential_offer = {
        "credential_issuer": cfgservice.service_url[:-1],
        "credential_configuration_ids": credentials_id,
        "grants": {"authorization_code": {}},
      }
[...]
```
with
```python
if "proceed" in form_keys:
  form = list(form_keys)
  form.remove("proceed")
  form.remove("credential_offer_URI")
  form.remove("Authorization Code Grant")
  selected_credential = request.form.get("credential_id")
  issuer_state_input = request.form.get("issuer_state", "").strip()

  if selected_credential and selected_credential in credentialsSupported:
    credentials_id = [selected_credential]
    session["credentials_id"] = credentials_id
    credentials_id_list = json.dumps(credentials_id)
    if auth_choice == "pre_auth_code":
      session["credential_offer_URI"] = credential_offer_URI
      return redirect(
        url_for("preauth.preauthRed", credentials_id=credentials_id_list)
      )

    else:
      grants = {"authorization_code": {}}
      if issuer_state_input:
        grants["authorization_code"]["issuer_state"] = issuer_state_input

      credential_offer = {
        "credential_issuer": cfgservice.service_url[:-1],
        "credential_configuration_ids": credentials_id,
        "grants": grants,
      }
[...]
```

### 2. Pull data from PID presentation

In `route_oid4vp.py`, make sure, you import the following:
```python
from urllib.parse import urlparse, parse_qs
import jwt
from .route_dynamic import dc4eu_api_call
```
Then, below
```python
for doctype in mdoc_json:
  for attribute, value in mdoc_json[doctype]:
    if attribute in attributesForm:
      attributesForm[attribute]["filled_value"] = value
    elif attribute in attributesForm2:
      attributesForm2[attribute]["filled_value"] = value
```
we add
```python
jws = session["authorization_params"]["token"]
claims = jwt.decode(
  jws,
  options={"verify_signature": False}  # skip signature check
)
query_str = claims.get("query", "")
params = parse_qs(query_str)

if "issuer_state" in params and params["issuer_state"]:
  collect_id = params["issuer_state"][0]

  if params["scope"][0] == "eu.europa.ec.eudi.ehic_drv_mdoc":
    document_type = "EHIC"
  elif params["scope"][0] == "eu.europa.ec.eudi.pda1_drv_mdoc":
    document_type = "PDA1"

  print("\n\nMDOC_JSON:", mdoc_json)

  data = {
    "collect_id": collect_id,
    "document_type": document_type,
    "birth_date": mdoc_json['eu.europa.ec.eudi.pid.1'][0][1],
    "authentic_source_person_id": mdoc_json['eu.europa.ec.eudi.pid.1'][1][1],
    "family_name": mdoc_json['eu.europa.ec.eudi.pid.1'][2][1],
    "given_name": mdoc_json['eu.europa.ec.eudi.pid.1'][3][1],
    "issuing_country": mdoc_json['eu.europa.ec.eudi.pid.1'][4][1],
    "nationality": mdoc_json['eu.europa.ec.eudi.pid.1'][5][1][0]
  }
  return dc4eu_api_call(data)
```

### 3. Add API call

Add the following function to the end of `route_dynamic.py`:
```python
def dc4eu_api_call(form_data):
    """Handles the API call to the dc4eu datastore"""

    api_endpoint = "http://your.local.ip.address:9080/api/v1/document/collect_id"
    
    # If "issuing_country" is present in "form_data.keys()", PID authentication was used. 
    # We can use the information in "form_data" to populate the payload.
    # Note, we assume that the PID contains a field "authentic_source_person_id".
    if "issuing_country" in form_data.keys():
        session["country"] = form_data["issuing_country"]
        payload = {
            "authentic_source": "authentic_source_" + form_data.get("nationality").lower(),
            "collect_id": form_data.get("collect_id"),
            "document_type": form_data.get("document_type"),
            "identity": {
                "authentic_source_person_id": form_data.get("authentic_source_person_id"),
                "birth_city": "",
                "birth_country": "",
                "birth_date": form_data["birth_date"],
                "birth_place": "",
                "birth_state": "",
                "family_name": form_data["family_name"],
                "family_name_at_birth": "",
                "gender": "",
                "given_name": form_data["given_name"],
                "given_name_at_birth": "",
                "nationality": "",
                "resident_address": "",
                "resident_city": "",
                "resident_country": "",
                "resident_house_number": "",
                "resident_postal_code": "",
                "resident_state": "",
                "resident_street": "",
                "schema": {
                    "name": "DefaultSchema",
                    "version": "1.0.0"
                }
            }
        }
    else:
        # Hardcoded Payload
        # Deprecated. Remained in the code, in case we want to revert to using formCountry.
        if form_data.get("collect_id") == "collect_id_10":
            payload = {
                "authentic_source": "authentic_source_se",
                "collect_id": form_data.get("collect_id"),
                "document_type": "EHIC",
                "identity": {
                    "authentic_source_person_id": "10",
                    "birth_city": "",
                    "birth_country": "",
                    "birth_date": "1970-01-10",
                    "birth_place": "",
                    "birth_state": "",
                    "family_name": "Castaneda",
                    "family_name_at_birth": "",
                    "gender": "",
                    "given_name": "Carlos",
                    "given_name_at_birth": "",
                    "nationality": "",
                    "resident_address": "",
                    "resident_city": "",
                    "resident_country": "",
                    "resident_house_number": "",
                    "resident_postal_code": "",
                    "resident_state": "",
                    "resident_street": "",
                    "schema": {
                        "name": "DefaultSchema",
                        "version": "1.0.0"
                    }
                }
            }
        if form_data.get("collect_id") == "collect_id_24":
            payload = {
                "authentic_source": "authentic_source_at",
                "collect_id": form_data.get("collect_id"),
                "document_type": "PDA1",
                "identity": {
                    "authentic_source_person_id": "24",
                    "birth_city": "",
                    "birth_country": "",
                    "birth_date": "1983-05-05",
                    "birth_place": "",
                    "birth_state": "",
                    "family_name": "Hoeger",
                    "family_name_at_birth": "",
                    "gender": "",
                    "given_name": "Hollis",
                    "given_name_at_birth": "",
                    "nationality": "",
                    "resident_address": "",
                    "resident_city": "",
                    "resident_country": "",
                    "resident_house_number": "",
                    "resident_postal_code": "",
                    "resident_state": "",
                    "resident_street": "",
                    "schema": {
                        "name": "DefaultSchema",
                        "version": "1.0.0"
                    }
                }
            }

    try:
        # Make the API call
        response = requests.post(api_endpoint, json=payload)
        response.raise_for_status()

        # Log success and return response to the frontend
        cfgserv.app_logger.info(f"API call successful: {response.json()}")

        api_data = response.json()
        document_data = api_data["data"]["document_data"]
        print("\n\nDOCUMENT-DATA:", document_data)
        if form_data.get("document_type") == "EHIC":
            mapped_form = {
                'subject' : [{
                    'family_name': document_data["subject"].get("family_name"),
                    'given_name': document_data["subject"].get("forename"),
                    'birth_date': document_data["subject"].get("date_of_birth"),
                }],
                'social_security_pin': document_data.get("social_security_pin"),
                'starting_date': document_data["period_entitlement"].get("starting_date"),
                'ending_date': document_data["period_entitlement"].get("ending_date"),
                'document_id': document_data.get("document_id"),
                'competent_institution' :[{
                    'institution_id': document_data["competent_institution"].get("institution_id"),
                    'institution_name': document_data["competent_institution"].get("institution_name"),
                    'country_code': document_data["competent_institution"].get("institution_country"),
                }],
                'version': '0.5',
                'issuing_country': 'FC',
                'issuing_authority': 'Test MDL issuer'
            }
        elif form_data.get("document_type") == "PDA1":
            list_details_of_employment = []

            for item in document_data["details_of_employment"]:
                temp_dict = {
                        'type_of_employment': item.get("type_of_employment"),
                        'name': item.get("name"),
                        'employer_id': item["ids_of_employer"][0].get("employer_id"),
                        'type_of_id': item["ids_of_employer"][0].get("type_of_id"),
                        'street': item["address"].get("street"),
                        'town': item["address"].get("town"),
                        'post_code': item["address"].get("post_code"),
                        'country': item["address"].get("country"),
                    }
                list_details_of_employment.append(temp_dict)

            list_places_of_work = []

            for item in document_data["places_of_work"]:
                list_place_of_work = []
                for item2 in item.get("place_of_work"):
                    temp_dict = {
                        'post_code': item2["address"].get("post_code"),
                        'street': item2["address"].get("street"),
                        'town': item2["address"].get("town"),
                        'company_vessel_name': item2.get("company_vessel_name"),
                        'flag_state_home_base': item2.get("flag_state_home_base"),
                        'company_id': item2["ids_of_company"][0].get("company_id"),
                        'type_of_id': item2["ids_of_company"][0].get("type_of_id"),
                    }
                    list_place_of_work.append(temp_dict)
                list_places_of_work.append({'country_work': item.get("country_work"), 'a_fixed_place_of_work_exist': item.get("a_fixed_place_of_work_exist"), "place_of_work": list_place_of_work})

            mapped_form = {
                'social_security_pin': document_data.get("social_security_pin"),
                'nationality': document_data.get("nationality"),
                'details_of_employment': list_details_of_employment,
                'places_of_work': list_places_of_work,
                'decision_legislation_applicable': [{
                    'member_state_which_legislation_applies': document_data["decision_legislation_applicable"].get("member_state_which_legislation_applies"),
                    'transitional_rule_apply': document_data["decision_legislation_applicable"].get("transitional_rule_apply"),
                    'starting_date': document_data["decision_legislation_applicable"].get("starting_date"),
                    'ending_date': document_data["decision_legislation_applicable"].get("ending_date"),
                }],
                'status_confirmation': document_data.get("status_confirmation"),
                'unique_number_of_issued_document': document_data.get("unique_number_of_issued_document"),
                'competent_institution' :[{
                    'institution_id': document_data["competent_institution"].get("institution_id"),
                    'institution_name': document_data["competent_institution"].get("institution_name"),
                    'country_code': document_data["competent_institution"].get("institution_country"),
                }],
                'version': '0.5',
                'issuing_country': 'FC',
                'issuing_authority': 'Test MDL issuer'
            }

        user_id = generate_unique_id()

        form_dynamic_data[user_id] = mapped_form.copy()
        form_dynamic_data[user_id].update({"expires": datetime.now() + timedelta(minutes=cfgserv.form_expiry)})

        if "jws_token" not in session or "authorization_params" in session:
            session["jws_token"] = session["authorization_params"]["token"]
        session["returnURL"] = cfgserv.OpenID_first_endpoint

        credentialsSupported = oidc_metadata["credential_configurations_supported"]

        presentation_data = dict()

        for credential_requested in session["credentials_requested"]:

            scope = credentialsSupported[credential_requested]["scope"]

            credential = credentialsSupported[credential_requested]["display"][0]["name"]

            presentation_data.update({credential: {}})

            credential_atributes_form = list()
            credential_atributes_form.append(credential_requested)
            attributesForm = getAttributesForm(credential_atributes_form).keys()
            attributesForm2 = getAttributesForm2(credential_atributes_form).keys()

            for attribute in mapped_form.keys():

                if attribute in attributesForm:
                    presentation_data[credential][attribute] = mapped_form[attribute]

                if attribute in attributesForm2:
                    presentation_data[credential][attribute] = mapped_form[attribute]

            if "issuer_config" in credentialsSupported[credential_requested]:
                doctype_config = credentialsSupported[credential_requested]["issuer_config"]

            today = date.today()
            expiry = today + timedelta(days=doctype_config["validity"])

            presentation_data[credential].update({"estimated_issuance_date": today.strftime("%Y-%m-%d")})
            presentation_data[credential].update({"estimated_expiry_date": expiry.strftime("%Y-%m-%d")})
            presentation_data[credential].update({"issuing_country": session["country"]}),
            presentation_data[credential].update({"issuing_authority": doctype_config["issuing_authority"]})

            if "credential_type" in doctype_config:
                presentation_data[credential].update({"credential_type": doctype_config["credential_type"]})

            if "birth_date" in presentation_data[credential] and "age_over_18" in presentation_data[credential]:
                presentation_data[credential].update({"age_over_18": True if calculate_age(
                    presentation_data[credential]["birth_date"]) >= 18 else False})

            if scope == "org.iso.18013.5.1.mDL":
                if "birth_date" in presentation_data[credential]:
                    presentation_data[credential].update({"age_over_18": True if calculate_age(
                        presentation_data[credential]["birth_date"]) >= 18 else False})

            if "driving_privileges" in presentation_data[credential]:
                json_priv = json.loads(presentation_data[credential]["driving_privileges"])
                presentation_data[credential].update({"driving_privileges": json_priv})

            if "portrait" in presentation_data[credential]:
                presentation_data[credential].update({"portrait": base64.b64encode(
                    base64.urlsafe_b64decode(presentation_data[credential]["portrait"])).decode("utf-8")})

            if "NumberCategories" in presentation_data[credential]:
                for i in range(int(presentation_data[credential]["NumberCategories"])):
                    f = str(i + 1)
                    presentation_data[credential].pop("IssueDate" + f)
                    presentation_data[credential].pop("ExpiryDate" + f)
                presentation_data[credential].pop("NumberCategories")

        print("\nPresentation_data: ", presentation_data)
        return render_template("dynamic/form_authorize.html", presentation_data=presentation_data,
                               user_id="FC." + user_id, redirect_url=cfgserv.service_url + "dynamic/redirect_wallet")

    except requests.exceptions.RequestException as e:
        # Log error and return failure response
        cfgserv.app_logger.error(f"EHIC API call failed: {str(e)}")
        return jsonify({"status": "error", "message": str(e)}), 500
```




