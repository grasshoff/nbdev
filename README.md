# Check Metadata and Upload to Zenodo
> This notebook will check the metadata according to certain norms and if they are correct, it will upload the data to Zenodo.


## Packages

```python
import yaml
from pathlib import Path
import pandas as pd
import sys
import requests
from IPython.display import display, Markdown
```

## Definitions

```python
def openfile(f):
    ending = f.suffix.replace('.', '')
    path = str(f)
    if ending == 'json':
        try:
            d = pd.read_json(path, orient = 'table')
        except:
            d = pd.read_json(path)
    if ending == 'csv':
        try:
            d = pd.read_csv(path, delimiter = ',')
        except:
            try:
                d = pd.read_csv(path, delimiter = '\t')
            except:
                d = pd.read_csv(path, delimiter = ';')
    if ending == 'yaml' or ending == 'cite':
        with open(path) as yml:
            d = yaml.load(yml, Loader=yaml.FullLoader)
    if ending == 'md':
        with open(path, encoding="utf8") as file:
            d = file.read()
            display(Markdown(d))
    return d

def findkeys(c):
    ck = list(c.keys())
    mk = list(c['Metadata'].keys())
    prk = [list(i.keys()) for i in c['Resources']]
    rk = [list(i.keys()) for i in c['Resources']][0]
    ak = [list(i['Attributes'].keys()) for i in c['Resources']]
    return ck, mk, prk, rk, ak

def checkVal(c, mk, ak):
    mke = []
    for i in mk:
        if c['Metadata'][i].strip() == '':
            print('Key {} may not be empty!'.format(i))
            mke.append(i)
    ake = []
    for i in ak:
        a = []
        for j in range(len(i)):
            if i[j].strip() == '':
                print('Key {} may not be empty!'.format(i[j]))
                mke.append(i[j])
    if not (mke or ake):
        message = None
        print('All metadata keys and attributes are filled, wonderful!')
    else:
        message = 'bad'
    return mke, ake, message

def checkkeys(pk, k):
    ke = []
    for i in pk:
        if i not in k:
            message = 'bad'
            print('The key {} does not exist in your metadata, please add it!'.format(i))
            ke.append(i)
        else:
            message = None
    return ke, message

def congrat(c1, c2, c3):
    if not (c1 or c2 or c3):
        message = None
        print('Congrats, all keys are set!')
    else:
        message = 'bad'
    return message

def comparekeys(allfn, cf, caks, dataDFkeys):
    if allfn != cf:
        missf = cf - allfn
        addedf = allfn - cf
        missk = []
        addedk = []
        messages = []
        print('You did not commented the file {} in your cite! Please to so!'.format(addedf))
        print('The file {} is not in the directory, please remove it from the cite!'.format(missf))
        messages.append('bad')
    else:
        missf = None
        addedf = None
        missk = []
        addedk = []
        allfn = list(allfn)
        messages = []
        for i in range(len(allfn)):
            print(allfn[i])
            print(i)
            if caks[i] != dataDFkeys[i]:
                misski = dataDFkeys[i] - caks[i]
                addedki = caks[i] - dataDFkeys[i]
                messages.append('bad')
                print('You did not commented the key {} in your cite! Please to so!'.format(misski))
                print('The key {} is not your data, please remove it from the cite!'.format(addedki))
                missk.append(misski)
                addedk.append(addedki)
        print(missk)
    if len(messages) == 0:
        message = None
    else:
        message = 'bad'
                
    return missf, addedf, missk, addedk, message
```

```python
def makeEmptyUpload(sandbox, ACCESS_TOKEN):
    
    # Create empty upload first to get the bucket_url
    headers = {"Content-Type": "application/json"}
    params = {'access_token': ACCESS_TOKEN}
    if sandbox:
        r = requests.post('https://sandbox.zenodo.org/api/deposit/depositions', params=params, json={}, headers=headers)
    else:
        r = requests.post('https://zenodo.org/api/deposit/depositions', params=params, json={}, headers=headers)
    print(r.status_code)
    bucket_url = r.json()["links"]["bucket"]
    return bucket_url, params

def uploadOneFile(bucket_url, params, filepath):

    # Give file
    p = Path(filepath)
    file = p.open("rb")
    filename = p.name
    
    # Upload file
    r = requests.put("%s/%s" % (bucket_url, filename), data=file, params=params)
    
    return r.json()
    
def uploadDirectory(bucket_url, params, dirpath):

    # Find all files in directory
    
    p = Path(dirpath)
    allfil = list(p.glob('*'))
    allfiles = [i.open("rb") for i in allfil]
    allfilenames = [i.name for i in allfil]
    
    # Upload files
    rs = []
    for i in range(len(allfiles)):
        r = requests.put("%s/%s" % (bucket_url, allfilenames[i]), data=allfiles[i], params=params)
        rs.append(r.json())
    return rs
```

## publprofil

```python
with open('./data/publprofil.yaml') as yml:
    pp = yaml.load(yml, Loader=yaml.FullLoader)
pp = pp['ResearchObject']
```

```python
pck, pmk, pprk, prk, pak = findkeys(pp)
```

## Dataset

```python
p = Path('./data/ModernLocationsIberia')
```

```python
allfile = list(p.glob('*'))
```

```python
allfn = set([i.name for i in allfile if not i.suffix == '.cite' and not i.suffix == '.md'])
```

### Cite

```python
c = list(sorted([i for i in allfile if i.suffix == '.cite']))[0]
```

```python
cite = openfile(c)['ResearchObject']
```

```python
cck, cmk, cprk, crk, cak = findkeys(cite)
```

```python
caks = [set(i) for i in cak]
```

```python
try:
    cf = set([i['File'.lower()] for i in cite['Resources']])
except:
    cf = set([i['File'] for i in cite['Resources']])
```

### Data

```python
d = list(sorted([i for i in allfile if not i.suffix == '.cite']))
```

```python
dataDFkeys = [set(openfile(f).keys()) for f in d if f.suffix == '.json' or f.suffix == '.csv']
```

#### Check Metadata keys & Attributes are not empty:

```python
cmke, cake, message1 = checkVal(cite, pmk, pak)
```

    All metadata keys and attributes are filled, wonderful!
    

#### Check existence of all keys:

```python
c1 = checkkeys(pck, cck)
c2 = checkkeys(pmk, cmk)
c3 = []
for i in cprk:
    c3.append(checkkeys(prk, i))
message2 = congrat(c1, c2, c3)
```

#### If data is identical to cite:

```python
missf, addedf, missk, addedk, message3 = comparekeys(allfn, cf, caks, dataDFkeys)
```

    You did not commented the file {'ModernLocationsIberia.csv'} in your cite! Please to so!
    The file {'ModernLocationsIberia.xlsx'} is not in the directory, please remove it from the cite!
    

## Upload on Zenodo

If there is any error in the cite, the process is terminated; otherwise one proceed with upload.

```python
if (message1 or message2 or message3):
    sys.exit()
```


    An exception has occurred, use %tb to see the full traceback.
    

    SystemExit
    


sandbox (for testing purposes) = sandbox ACCESS_TOKEN from https://sandbox.zenodo.org/account/settings/applications/tokens/new/

real data (for real upload) = ACCESS_TOKEN from https://zenodo.org/account/settings/applications/tokens/new/

```python
''' 
    Register for a Zenodo sandbox account if you don't already have one.
    Go to https://sandbox.zenodo.org/account/settings/applications/tokens/new/.
    Select the OAuth scopes you need (for the quick start tutorial you need deposit:write and deposit:actions).
    Please insert your just generated token for ... below.
'''

ACCESS_TOKEN = '...'
```

ONLY CHANGE SANDBOX TO FALSE IF YOU KNOW WHAT YOU ARE DOING, THIS WILL BE A REAL UPLOAD THEN! If you want to test something always use sandbox!

```python
bucket_url, params = makeEmptyUpload(True, ACCESS_TOKEN)
```

    201
    

If you get a number lower than 400 everything is fine!

```python
uploadDirectory(bucket_url, params, str(p.resolve()))
```




    [{'mimetype': 'application/octet-stream',
      'updated': '2020-12-16T15:43:26.187743+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/a4eef343-e449-400c-84e2-f1c8e9b29c39/ModernLocationsDocu.md',
       'version': 'https://sandbox.zenodo.org/api/files/a4eef343-e449-400c-84e2-f1c8e9b29c39/ModernLocationsDocu.md?versionId=ca0bbff7-07de-498a-b61d-05ba15193f4f',
       'uploads': 'https://sandbox.zenodo.org/api/files/a4eef343-e449-400c-84e2-f1c8e9b29c39/ModernLocationsDocu.md?uploads'},
      'is_head': True,
      'created': '2020-12-16T15:43:26.182095+00:00',
      'checksum': 'md5:a5351624ad3e1a780108ed6d88b510fe',
      'version_id': 'ca0bbff7-07de-498a-b61d-05ba15193f4f',
      'delete_marker': False,
      'key': 'ModernLocationsDocu.md',
      'size': 734},
     {'mimetype': 'application/octet-stream',
      'updated': '2020-12-16T15:43:26.700750+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/a4eef343-e449-400c-84e2-f1c8e9b29c39/ModernLocationsIberia.cite',
       'version': 'https://sandbox.zenodo.org/api/files/a4eef343-e449-400c-84e2-f1c8e9b29c39/ModernLocationsIberia.cite?versionId=d2aed9ae-98a6-487d-afe5-3a6fcfa603d7',
       'uploads': 'https://sandbox.zenodo.org/api/files/a4eef343-e449-400c-84e2-f1c8e9b29c39/ModernLocationsIberia.cite?uploads'},
      'is_head': True,
      'created': '2020-12-16T15:43:26.696147+00:00',
      'checksum': 'md5:d0fb478e1e5bb6d886217dd16396eef3',
      'version_id': 'd2aed9ae-98a6-487d-afe5-3a6fcfa603d7',
      'delete_marker': False,
      'key': 'ModernLocationsIberia.cite',
      'size': 1171},
     {'mimetype': 'text/csv',
      'updated': '2020-12-16T15:43:27.272359+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/a4eef343-e449-400c-84e2-f1c8e9b29c39/ModernLocationsIberia.csv',
       'version': 'https://sandbox.zenodo.org/api/files/a4eef343-e449-400c-84e2-f1c8e9b29c39/ModernLocationsIberia.csv?versionId=ccaa70b9-c22a-4f01-bdae-a848011e94aa',
       'uploads': 'https://sandbox.zenodo.org/api/files/a4eef343-e449-400c-84e2-f1c8e9b29c39/ModernLocationsIberia.csv?uploads'},
      'is_head': True,
      'created': '2020-12-16T15:43:27.267774+00:00',
      'checksum': 'md5:cfc425f53644b6e9a587e32986c3bf57',
      'version_id': 'ccaa70b9-c22a-4f01-bdae-a848011e94aa',
      'delete_marker': False,
      'key': 'ModernLocationsIberia.csv',
      'size': 22730}]


