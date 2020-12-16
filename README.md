# Check Metadata and Upload to Zenodo
> This notebook will check the metadata according to certain norms and if they are correct, it will upload the data to Zenodo.


## Packages

```python
import yaml
from pathlib import Path
import pandas as pd
import sys
import requests
```

## Definitions

```python
def openfile(f):
    ending = str(f).split('.')[-1]
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
            d = pd.read_csv(path, delimiter = '\t')
    if ending == 'yaml' or ending == 'cite':
        with open(path) as yml:
            d = yaml.load(yml, Loader=yaml.FullLoader)
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
        message = 'bad'
        print('You did not commented the file {} in your cite! Please to so!'.format(missf))
        print('The file {} is not in the directory, please remove it from the cite!'.format(addedf))
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
p = Path('./data/Parapegmata')
```

```python
allfile = list(p.glob('*'))
```

```python
allfn = set([i.name for i in allfile if not i.suffix == '.cite'])
```

### Cite

```python
c = list(sorted([i for i in allfile if str(i).split('.')[-1] == 'cite']))[0]
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
cf = set([i['File'.lower()] for i in cite['Resources']])
```

### Data

```python
d = list(sorted([i for i in allfile if not i.suffix == '.cite']))
```

```python
dataDFkeys = [set(openfile(f).keys()) for f in d]
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

    The key File does not exist in your metadata, please add it!
    The key File does not exist in your metadata, please add it!
    The key File does not exist in your metadata, please add it!
    The key File does not exist in your metadata, please add it!
    The key File does not exist in your metadata, please add it!
    The key File does not exist in your metadata, please add it!
    The key File does not exist in your metadata, please add it!
    The key File does not exist in your metadata, please add it!
    

#### If data is identical to cite:

```python
missf, addedf, missk, addedk, message3 = comparekeys(allfn, cf, caks, dataDFkeys)
```

    Milet.json
    0
    You did not commented the key {'addition_ID', 'zodiac_part_ID', 'feast_ID', 'feast', 'ID', 'text_string', 'feast_greek', 'addition', 'addition_greek', 'zodiac_part'} in your cite! Please to so!
    The key {'parallel', 'meteo_event_class_ID', 'parallel_ID', 'authority', 'meteo_event_class', 'record_ID', 'authority_ID'} is not your data, please remove it from the cite!
    Madrid.json
    1
    Paris.json
    2
    You did not commented the key {'day_length_fractions', 'month', 'length_month', 'meteo_addition_text_string', 'night_length_fractions', 'zodiac_part', 'meteo_statement', 'day_length', 'night_length_greek', 'day', 'column', 'addition_text_string_greek', 'season', 'zodiac_part_ID', 'season_greek', 'day_length_greek', 'fragment', 'night_length_footnote', 'addition_text_string', 'text_passage', 'night_length', 'day_length_footnote', 'month_ID', 'meteo_addition_text_string_greek', 'season_ID'} in your cite! Please to so!
    The key {'authority_ID_Meteo', 'meteo_event_class_ID', 'authority_Meteo', 'hole_type_ID', 'hole_No', 'meteo_event_class', 'fragment_ID', 'hole_type', 'authority_ID_Astro', 'authority_Astro'} is not your data, please remove it from the cite!
    Geminos.json
    3
    You did not commented the key {'supplement_Meteo', 'supplement_ID'} in your cite! Please to so!
    The key {'addition_ID', 'zodiac_part_ID', 'feast_ID', 'feast', 'feast_greek', 'addition', 'addition_greek', 'zodiac_part'} is not your data, please remove it from the cite!
    Oxford.json
    4
    You did not commented the key {'authority_ID_Meteo', 'meteo_event_class_ID', 'authority_Meteo', 'hole_type_ID', 'hole_No', 'meteo_event_class', 'fragment_ID', 'hole_type', 'authority_ID_Astro', 'authority_Astro'} in your cite! Please to so!
    The key {'day_length_fractions', 'month', 'length_month', 'meteo_addition_text_string', 'night_length_fractions', 'zodiac_part', 'meteo_statement', 'day_length', 'night_length_greek', 'day', 'column', 'addition_text_string_greek', 'season', 'zodiac_part_ID', 'season_greek', 'day_length_greek', 'fragment', 'type', 'night_length_footnote', 'addition_text_string', 'night_length', 'text_passage', 'day_length_footnote', 'month_ID', 'meteo_addition_text_string_greek', 'season_ID'} is not your data, please remove it from the cite!
    Hibeh.json
    5
    You did not commented the key {'feast_ID', 'feast', 'meteo_event_class_ID', 'meteo_event_class', 'feast_greek'} in your cite! Please to so!
    The key {'text_string', 'supplement_Meteo', 'supplement_ID'} is not your data, please remove it from the cite!
    Antiochos.json
    6
    You did not commented the key {'addition_ID', 'status', 'supplement_ID', 'Authority_ID', 'authority', 'addition', 'addition_greek'} in your cite! Please to so!
    The key {'feast_ID', 'feast', 'feast_greek'} is not your data, please remove it from the cite!
    Phaseis.json
    7
    You did not commented the key {'parallel_ID', 'parallel', 'authority_ID', 'record_ID'} in your cite! Please to so!
    The key {'addition_ID', 'status', 'ID', 'supplement_ID', 'Authority_ID', 'addition', 'addition_greek'} is not your data, please remove it from the cite!
    

## Upload on Zenodo

If there is any error in the cite, the process is terminated; otherwise one proceed with upload.

```python
if (message1 or message2 or message3):
    sys.exit()
```


    An exception has occurred, use %tb to see the full traceback.
    

    SystemExit
    


    C:\Users\Public\Anaconda\lib\site-packages\IPython\core\interactiveshell.py:3334: UserWarning: To exit: use 'exit', 'quit', or Ctrl-D.
      warn("To exit: use 'exit', 'quit', or Ctrl-D.", stacklevel=1)
    

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




    [{'mimetype': 'application/json',
      'updated': '2020-12-16T10:46:10.461734+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Antiochos.json',
       'version': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Antiochos.json?versionId=fb16f68e-5b7a-4a53-912e-f4480ecb44d1',
       'uploads': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Antiochos.json?uploads'},
      'is_head': True,
      'created': '2020-12-16T10:46:10.455055+00:00',
      'checksum': 'md5:9f8b8b88c310dc7256f23ae114eecb10',
      'version_id': 'fb16f68e-5b7a-4a53-912e-f4480ecb44d1',
      'delete_marker': False,
      'key': 'Antiochos.json',
      'size': 78439},
     {'mimetype': 'application/json',
      'updated': '2020-12-16T10:46:10.998593+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Geminos.json',
       'version': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Geminos.json?versionId=5e82321a-aa1b-4ecb-b1a9-eaa7d1fc26ee',
       'uploads': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Geminos.json?uploads'},
      'is_head': True,
      'created': '2020-12-16T10:46:10.993280+00:00',
      'checksum': 'md5:84278864159cdef63ed7f6e4dca997a5',
      'version_id': '5e82321a-aa1b-4ecb-b1a9-eaa7d1fc26ee',
      'delete_marker': False,
      'key': 'Geminos.json',
      'size': 303197},
     {'mimetype': 'application/json',
      'updated': '2020-12-16T10:46:11.297076+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Hibeh.json',
       'version': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Hibeh.json?versionId=0169900c-86ef-4d53-a9ec-77764f10ce39',
       'uploads': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Hibeh.json?uploads'},
      'is_head': True,
      'created': '2020-12-16T10:46:11.291805+00:00',
      'checksum': 'md5:8e0d25ec5398bf288250ed6892ba6615',
      'version_id': '0169900c-86ef-4d53-a9ec-77764f10ce39',
      'delete_marker': False,
      'key': 'Hibeh.json',
      'size': 46200},
     {'mimetype': 'application/json',
      'updated': '2020-12-16T10:46:11.642247+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Madrid.json',
       'version': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Madrid.json?versionId=5524dfd6-1ad1-479a-99aa-4dc5a923bc3b',
       'uploads': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Madrid.json?uploads'},
      'is_head': True,
      'created': '2020-12-16T10:46:11.634381+00:00',
      'checksum': 'md5:7c700e14d8d04d0041db949db451fb9b',
      'version_id': '5524dfd6-1ad1-479a-99aa-4dc5a923bc3b',
      'delete_marker': False,
      'key': 'Madrid.json',
      'size': 83574},
     {'mimetype': 'application/json',
      'updated': '2020-12-16T10:46:12.116721+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Milet.json',
       'version': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Milet.json?versionId=82f9cc11-ebee-42bf-94a5-afa42d74f5c3',
       'uploads': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Milet.json?uploads'},
      'is_head': True,
      'created': '2020-12-16T10:46:12.111106+00:00',
      'checksum': 'md5:b1ecb72148b8a5c3e878f3d074b8ad97',
      'version_id': '82f9cc11-ebee-42bf-94a5-afa42d74f5c3',
      'delete_marker': False,
      'key': 'Milet.json',
      'size': 28419},
     {'mimetype': 'application/json',
      'updated': '2020-12-16T10:46:12.404082+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Oxford.json',
       'version': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Oxford.json?versionId=669dd8e1-4a27-463a-9139-debd9e4119c7',
       'uploads': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Oxford.json?uploads'},
      'is_head': True,
      'created': '2020-12-16T10:46:12.399069+00:00',
      'checksum': 'md5:e69ecabd1b3049fa858178cd0a4bfe67',
      'version_id': '669dd8e1-4a27-463a-9139-debd9e4119c7',
      'delete_marker': False,
      'key': 'Oxford.json',
      'size': 33330},
     {'mimetype': 'application/octet-stream',
      'updated': '2020-12-16T10:46:12.656568+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Parapegmata.cite',
       'version': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Parapegmata.cite?versionId=cbe6751a-22fd-4692-824f-885ebe4a0f0d',
       'uploads': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Parapegmata.cite?uploads'},
      'is_head': True,
      'created': '2020-12-16T10:46:12.651910+00:00',
      'checksum': 'md5:ffe53c1cefcede6529615ca3ca6d7db9',
      'version_id': 'cbe6751a-22fd-4692-824f-885ebe4a0f0d',
      'delete_marker': False,
      'key': 'Parapegmata.cite',
      'size': 16270},
     {'mimetype': 'application/json',
      'updated': '2020-12-16T10:46:12.966468+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Paris.json',
       'version': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Paris.json?versionId=c27c8048-fa80-4674-a774-67812622a67d',
       'uploads': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Paris.json?uploads'},
      'is_head': True,
      'created': '2020-12-16T10:46:12.960431+00:00',
      'checksum': 'md5:6917abbae53d2669ea1e773cf272c0a4',
      'version_id': 'c27c8048-fa80-4674-a774-67812622a67d',
      'delete_marker': False,
      'key': 'Paris.json',
      'size': 127034},
     {'mimetype': 'application/json',
      'updated': '2020-12-16T10:46:13.290260+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Phaseis.json',
       'version': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Phaseis.json?versionId=45727d9b-dfe8-4e2d-9220-f6571d45a4ad',
       'uploads': 'https://sandbox.zenodo.org/api/files/8530cff0-7b19-4a56-a862-c047e5b32eed/Phaseis.json?uploads'},
      'is_head': True,
      'created': '2020-12-16T10:46:13.284010+00:00',
      'checksum': 'md5:33deed0fdb34e2ff81a5904430c7d1b1',
      'version_id': '45727d9b-dfe8-4e2d-9220-f6571d45a4ad',
      'delete_marker': False,
      'key': 'Phaseis.json',
      'size': 920426}]


