# Ferrit Instances

This repository maintains a list of [Ferrit](https://github.com/ferritreader/ferrit) instances in JSON format, providing the URL, location, and Ferrit version for each instance. A helper script exists in this repository to generate the list in JSON form.

## Contributing

To contribute new instances to this repo:

1. familiarize yourself with the repo 
  - learn its structure by reading the section [Contents](#contents)
  - learn how to generate the `instances.json` file in the sections [Adding or removing an instance](#adding-or-removing-an-instance) and [`generate-instances-json.sh`](#generate-instances-jsonsh)
1. [fork the repo](/fork), and in your fork make your additions to `instances.txt` and generate a new `instances.json`
1. [open a PR](/compare) from your fork to the main repo

## Contents

This repo consists of four files:

1. `instances.json`: This is the list of Ferrit instances.
1. `instances-schema.json`: JSON Schema governing `instances.json`.
1. `instances.txt`: This is a CSV of Ferrit instances. While this is also machine-readable, it is recommended to use `instances.json` instead. `instances.txt` is meant for contributors to add and remove instances, and `generate-instances-json.sh` will validate those instances and generate `instances.json`.
1. `generate-instances-json.sh`: This script takes in a CSV file as input, typically `instances.txt`, and outputs a JSON object with a list of Ferrit instances. This is the script that generates `instances.json`.

### Adding or removing an instance

To generate `instances.json`, perform the following:

1. Modify `instances.txt` to add or remove instances. See [Expected CSV format](#Expected CSV format) for the expected format of each CSV row.
1. Run `generate-instances-json.sh -i ./instances.txt -o ./instances.json` to generate `instances.json`. The existing `instances.json` will be replaced.

Pull requests to add or remove instances are always welcome.

### `generate-instances-json.sh`

`generate-instances-json.sh` is the script that produces a JSON of [Ferrit](https://github.com/ferritreader/ferrit) instances, given a CSV input of Ferrit instances.

Unless `-i` and `-o` are specified (see [Usage](#Usage) below), the input and output are assumed the stdin and stdout streams respectively.

#### Usage

```
USAGE
    ./generate-instances-json.sh [-I INPUT_JSON] [-T] [-f] [-i INPUT_CSV] [-o OUTPUT_JSON]
    ./generate-instances-json.sh -h

DESCRIPTION
    Generate a JSON of Ferrit instances, given a CSV input listing those
    instances.

    The INPUT_CSV file must be a file in CSV syntax of the form

        [url],[country code],[cloudflare enabled],[description]

    where all four parameters are required (though the description may be
    blank). Except for onion and I2P sites, all URLs MUST be HTTPS.

    OUTPUT_JSON will be overwritten if it exists. No confirmation will be
    requested from the user.

    By default:

    * This script will not attempt to connect to I2P instances. If you want
      this script to consider instances on the I2P network, you will need to
      provide an HTTP proxy in the environment variable I2P_HTTP_PROXY.
      This proxy typically listens at 127.0.0.1:4444.

    * This script will attempt to connect to instances in the CSV that are on
      Tor, provided that it can (it will check to see if Tor is running and the
      availability of the torsocks program). If you want to disable connections
      to these onion sites, provide the -T option.

OPTIONS
    -I INPUT_JSON
        Import the list of Ferrit onion and I2P instances from the file
        INPUT_JSON. To use stdin, provide `-I -`. Implies -T, and further
        causes the script to ignore the value in I2P_HTTP_PROXY. Note that the
        argument provided to this option CANNOT be the same as the argument
        provided to -i. If the JSON could not be read, the script will exit with
        status code 1.

    -T
        Do not connect to Tor. Onion sites in INPUT_CSV will not be processed.
        Assuming no other failure, the script will still exit with status code
        0.

    -f
        Force the script to exit, with status code 1, upon the first failure to
        connect to an instance. Normally, the script will continue to build and
        output the JSON even when one or more of the instances could not be
        reached, though the exit code will be non-zero.

    -i INPUT_CSV
        Use INPUT_CSV as the input file. To read from stdin (the default
        behavior), either omit this option or provide `-i -`. Note that the
        argument provided to this option CANNOT be the same as the argument
        provided to -I.

    -o OUTPUT_JSON
        Write the results to OUTPUT_JSON. Any existing file will be
        overwritten. To write to stdout (the default behavior), either omit
        this option or provide `-o -`.

ENVIRONMENT

    USER_AGENT
        Sets the User-Agent that curl will use when making the GET to each
        website. By default, this script will tell curl to set its User-Agent
        string to "ferrit-instance-updater/0.1".

    I2P_HTTP_PROXY
        HTTP proxy for connecting to the I2P network. This is required in
        order to connect to instances on I2P. If -I is provided, the value in
        this variable is ignored.
```

#### Prerequisites

`generate-instances-json.sh` requires **curl** in order to make HTTP(S) requests and **jq** to process and format JSON.

**tor** and **torsocks** are required for processing onion sites, but the script will skip instances on Tor if neither tor is running nor torsocks is available. An option exists to import onion sites from an existing JSON file should you wish not to use tor.

#### Expected CSV format

The CSV must take on the form:

```
[url],[country code],[cloudflare enabled],[description]
```

Each field described:
- **url** (REQUIRED): The url to the Ferrit instance. This _must_ be HTTPS, unless the instance is an onion/I2P site.
- **country code** (REQUIRED): The two-letter code for the country in which the instance is hosted, in caps.
- **cloudflare enabled** (REQUIRED): A boolean; true if the instance sits behind Cloudflare.
- **description** (REQUIRED): A description of the instance; a description can be blank, but one must be provided for the script to parse the CSV correctly. **As this description string becomes a JSON value without any transformation, any special characters, including and especially newlines, must be escaped.**

#### Processing the CSV

The script will process the CSV and for each row connect to the URL and get the version string of the running instance. For each row, if the connection is successful and the script can determine the version, it will yield a JSON object (an "entry") of the form:

```json
{
    "url": "<url>",
    "version": "<version>",
    "cloudflare": <true if cloudflare is enabled; null otherwise>,
    "description: "<description if non-empty; null otherwise>"
}
```

At the end, the script will assemble the entries into a JSON array and place them in a new JSON object:

```json
{
    "updated": "<today's date (at the Greenwich meridian) in ISO 8601 format>",
    "instances: [<entries>]
}
```

If all instances could be processed, the script exits with an exit code of 0. If the script was unable to process an instance, it will continue processing other instances, but the exit code will be 1. If there was an error to do with processing the CSV, the exit code is 2.

#### Instances on Tor or I2P

This script will attempt to connect to instances that are onion or I2P sites. 

##### Tor

To make sure it can connect to onion sites, the script will see if Tor is running and if torsocks is installed. If neither condition is met, the script will not attempt to connect to Ferrit onion sites and will skip them. The exit code will still be 0, assuming that the WWW Ferrit sites were processed without error.

##### I2P

In order to allow the script to connect to I2P, you must specify a proxy host and port in the environment variable `I2P_HTTP_PROXY`. This is typically 127.0.0.1:4444, unless your proxy listens on a separate address and/or port. If this environment variable is not defined or it is empty, the script will not attempt to connect to Ferrit I2P sites and will skip them. The exit code will still be 0, assuming that the WWW Ferrit sites were processed without error.

## License

The script `generate-instances-json.sh` and the schema file `instances-schema.json` are licensed under [the GNU General Public License v3.0](https://www.gnu.org/licenses/gpl-3.0.en.html). `instances.json` and `instances.txt` are released to the public domain.
