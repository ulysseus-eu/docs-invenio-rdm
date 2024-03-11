# How to implement the project

## 1. Download all GitHub repositories

- [invenio-communities](https://github.com/ulysseus-eu/invenio-communities)
- [invenio-app-rdm](https://github.com/ulysseus-eu/invenio-app-rdm)
- [invenio-rdm-records](https://github.com/ulysseus-eu/invenio-rdm-records)

## 2. Creating a new instance of invenioRDM

Refer to the [official documentation](https://inveniordm.docs.cern.ch/install/) for installation instructions.

## 3. Editing the Pipfile

Add your version of invenioRDM and correct repository paths:

```bash
[packages]
invenio-communities = {path="../invenio-communities"}
invenio-rdm-records = {path="../invenio-rdm-records"}
invenio-app-rdm = {path="../invenio-app-rdm", extras = ["postgresql"], version = "~=12.0.0b2.dev0"}
instance-dev = {editable="True", path="./site"}
```

## 4. Removing dependencies from locally cloned repositories

In the setup.cfg of **invenio-app-rdm**, delete:

```bash
invenio-communities>=10.0.0,<11.0.0
invenio-rdm-records>=6.0.0,<7.0.0
```

In the setup.cfg of **invenio-rdm-records**, delete:

```plaintext
invenio-communities>=10.0.0,<11.0.0
```

## 5. Delete pipfile.lock

## 6. Switch to the 'person' branch before installation:

```bash
invenio-cli install
```

## 7. Troubleshooting

|Error|Solution|
|---|---|
|react-invenio-forms|Go to the webpack.py file and standardize the values of "react-invenio-forms" for all the repositories|


