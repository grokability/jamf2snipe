# jamf2snipe
## Import/Sync Computers from JAMF to Snipe-IT
```
usage: jamf2snipe [-h] [-v] [--auto_incrementing] [--dryrun] [-d] [--do_not_update_jamf] [--do_not_verify_ssl] [-r] [-f] [--version] [-u | -ui | -uf] [-uns] [-m | -c]

options:
  -h, --help            show this help message and exit
  -v, --verbose         Sets the logging level to INFO and gives you a better
                        idea of what the script is doing.
  --auto_incrementing   You can use this if you have auto-incrementing
                        enabled in your snipe instance to utilize that
                        instead of adding the Jamf ID for the asset tag.
  --dryrun              This checks your config and tries to contact both
                        the JAMFPro and Snipe-it instances, but exits before
                        updating or syncing any assets.
  -d, --debug           Sets logging to include additional DEBUG messages.
  --do_not_update_jamf  Does not update Jamf with the asset tags stored in
                        Snipe.
  --do_not_verify_ssl   Skips SSL verification for all requests. Helpful when
                        you use self-signed certificate.
  -r, --ratelimited     Puts a half second delay between API calls to adhere
                        to the standard 120/minute rate limit
  -f, --force           Updates the Snipe asset with information from Jamf
                        every time, despite what the timestamps indicate.
  --version             Prints the version and exits.
  -u, --users           Checks out the item to the current user in Jamf if
                        it's not already deployed
  -ui, --users_inverse  Checks out the item to the current user in Jamf if
                        it's already deployed
  -uf, --users_force    Checks out the item to the user specified in Jamf no
                        matter what
  -uns, --users_no_search
                        Doesn't search for any users if the specified fields
                        in Jamf and Snipe don't match. (case insensitive)
  -m, --mobiles         Runs against the Jamf mobiles endpoint only.
  -c, --computers       Runs against the Jamf computers endpoint only.
```

## Overview:
This tool will sync assets between a JAMF Pro instance and a Snipe-IT instance. The tool searches for assets based on the serial number, not the existing asset tag. If assets exist in JAMF and are not in Snipe-IT, the tool will create an asset and try to match it with an existing Snipe model. This match is based on the Mac's model identifier (ex. MacBookAir7,1) being entered as the model number in Snipe, rather than the model name. If a matching model isn't found, it will create one.

When an asset is first created, it will fill out only the most basic information. When the asset already exists in your Snipe inventory, the tool will sync the information you specify in the settings.conf file and make sure that the asset_tag field in JAMF matches the asset tag in Snipe, where Snipe's info is considered the authority.

> Because it determines whether JAMF or Snipe has the most recently updated record, there is the potential to have blank data in Jamf overwrite good data in Snipe (ex. purchase date).

Lastly, if the asset_tag field is blank in JAMF when it is being created in Snipe, then the tool will look for a 4 or more digit number in the computer name. If it fails to find one, it will use JAMFID-<jamfid#> as the asset tag in Snipe. This way, you can easily filter this out and run scripts against it to correct in the future.

## Requirements:

- Python3 (3.7 or higher) is installed on your system with the requests, json, time, and configparser python libs installed.
- Network access to both your JAMF and Snipe-IT environments.
- A JAMF client_id and client_secret for the JAMF API that has read & write permissions for computer assets, mobile device assets, and users.
  - Computers: Read, Update
  - Mobile Devices: Read, Update
  - Users: Read, Update
- Snipe API key for a user that has edit/create permissions for assets and models. Snipe-IT documentation instructions for creating API keys: [https://snipe-it.readme.io/reference#generating-api-tokens](https://snipe-it.readme.io/reference#generating-api-tokens)

## Installation:

### Mac

1. Install Python 3.7 or later
  - Grab the latest PKG installer from the Python website and run it.

2. Add Python 3.7 or later to your PATH
  - Run the `Update Shell Profile.command` script in the `/Applications/Python 3.X` folder to add `python3.X` your PATH

3. Create a virtualenv for jamf2snipe
  - Create the virtualenv: `mkdir ~/.virtualenv`
  - Add `python3.X` to the virtualenv: `python3.X -m venv ~/.virtualenv/jamf2snipe`
  - Activate the virtualenv: `source ~/.virtualenv/jamf2snipe/bin/activate`

4. Install dependencies
  - `pip install -r /path/to/jamf2snipe/requirements.txt`

5. Configure settings.conf (you can start by copying settings.conf.example to settings.conf)
6. Run `python jamf2snipe` & profit

### Linux

1. Copy the files to your system (recommend installing to /opt/jamf2snipe/* ). Make sure you meet all the system requirements.
2. Edit the settings.conf to match your current environment - you can start by copying settings.conf.example to settings.conf. The script will look for a valid settings.conf in /opt/jamf2snipe/settings.conf, /etc/jamf2snipe/settings.conf, or in the current folder (in that order): so either copy the file to one of those locations, or be sure that the user running the program is in the same folder as the settings.conf.

## Configuration - settings.conf:

All of the settings that are listed in the [settings.conf.example](https://github.com/grokability/jamf2snipe/blob/main/settings.conf.example) are required except for the api-mapping section. It's recommended that you install these files to /opt/jamf2snipe/ and run them from there. You will need valid subsets from [JAMF's API](https://developer.jamf.com/apis/classic-api/index) to associate fields into Snipe.

### Required

Note: do not add `""` or `''` around any values.

**[jamf]**

- `url`: https://*your_jamf_instance*.com:*port*
- `client_id`: Jamf API client ID
- `client_secret`: Jamf API client secret

**[snipe-it]**

Check out the [settings.conf.example](https://github.com/grokability/jamf2snipe/blob/main/settings.conf.example) file for the full documentation

- `url`: http://*your_snipe_instance*.com
- `apikey`: API key generated via [these steps](https://snipe-it.readme.io/reference#generating-api-tokens).
- `manufacturer_id`: The manufacturer database field id for the Apple in your Snipe-IT instance. You will probably have to create a Manufacturer in Snipe-IT and note its ID.
- `defaultStatus`: The status database field id to assign to any assets created in Snipe-IT from JAMF. Usually you will want to pick a status like "Ready To Deploy" - look up its ID in Snipe-IT and put the ID here.
- `computer_model_category_id`: The ID of the category you want to assign to JAMF computers. You will have to create this in Snipe-IT and note the Category ID
- `mobile_model_category_id`: The ID of the category you want to assign to JAMF mobile devices. You will have to create this in Snipe-IT and note the Category ID

### API Mapping

To get the database fields for Snipe-IT Custom Fields, go to Custom Fields, scroll down past Fieldsets to Custom Fields, click the column selection and button and select the unchecked 'DB Field' checkbox. Copy and paste the DB Field name for the Snipe under api-mapping in settings.conf.

To get the database fields for Jamf, refer to Jamf's ["Classic" API documentation](https://developer.jamf.com/apis/classic-api/index).

You need to set the manufacturer_id for Apple devices in the settings.conf file.  To get this, go to Manufacturers, click the column selection button and select the `ID` checkbox.

Some example API mappings can be found below:

- Computer Name:		`name = general name`
- MAC Address:		`_snipeit_mac_address_1 = general mac_address`
- IPv4 Address:		`_snipeit_<your_IPv4_custom_field_id> = general ip_address`
- Purchase Cost:		`purchase_cost = purchasing purchase_price`
- Purchase Date:		`purchase_date = purchasing po_date`
- OS Version:			`_snipeit_<your_OS_version_custom_field_id> = hardware os_version`
- Extension Attribute:    `_snipe_it_<your_custom_field_id> = extension_attributes <attribute id from jamf>`

More information can be found in the ./jamf2snipe file about associations and [valid subsets](https://github.com/ParadoxGuitarist/jamf2snipe/blob/master/jamf2snipe#L33).

## Testing

It is *always* a good idea to create a test environment to ensure everything works as expected before running anything in production.

Because `jamf2snipe` only ever writes the asset_tag for a matching serial number back to Jamf, testing with your production JAMF Pro is OK. However, this can overwrite good data in Snipe. You can spin up a Snipe instance in Docker pretty quickly ([see the Snipe docs](https://snipe-it.readme.io/docs/docker)).

## Contributing

Thanks to all of the people that have already contributed to this project! If you have something you'd like to add please help by forking this project then creating a pull request to the `devel` branch. When working on new features, please try to keep existing configs running in the same manner with no changes. When possible, open up an issue and reference it when you make your pull request.

Files are formatted with `black` formatter, please run:

```bash
pip install black && black jamf2snipe
```