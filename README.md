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
with open('./publprofil.yaml') as yml:
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

    Oxford.json
    0
    You did not commented the key {'addition_ID', 'addition_greek', 'feast', 'feast_greek', 'zodiac_part', 'ID', 'feast_ID', 'text_string', 'zodiac_part_ID', 'addition'} in your cite! Please to so!
    The key {'meteo_event_class', 'parallel', 'authority', 'meteo_event_class_ID', 'authority_ID', 'parallel_ID', 'record_ID'} is not your data, please remove it from the cite!
    Paris.json
    1
    Milet.json
    2
    You did not commented the key {'month_ID', 'night_length_greek', 'meteo_statement', 'day', 'text_passage', 'day_length_footnote', 'night_length_footnote', 'column', 'addition_text_string_greek', 'addition_text_string', 'night_length', 'day_length', 'zodiac_part', 'fragment', 'night_length_fractions', 'season', 'month', 'day_length_greek', 'season_ID', 'season_greek', 'day_length_fractions', 'meteo_addition_text_string_greek', 'meteo_addition_text_string', 'length_month', 'zodiac_part_ID'} in your cite! Please to so!
    The key {'authority_ID_Meteo', 'fragment_ID', 'meteo_event_class', 'hole_type_ID', 'authority_Meteo', 'meteo_event_class_ID', 'hole_type', 'authority_Astro', 'authority_ID_Astro', 'hole_No'} is not your data, please remove it from the cite!
    Phaseis.json
    3
    You did not commented the key {'supplement_Meteo', 'supplement_ID'} in your cite! Please to so!
    The key {'addition_ID', 'addition_greek', 'feast', 'feast_greek', 'zodiac_part', 'feast_ID', 'zodiac_part_ID', 'addition'} is not your data, please remove it from the cite!
    Geminos.json
    4
    You did not commented the key {'authority_ID_Meteo', 'fragment_ID', 'meteo_event_class', 'hole_type_ID', 'authority_Meteo', 'meteo_event_class_ID', 'hole_type', 'authority_Astro', 'authority_ID_Astro', 'hole_No'} in your cite! Please to so!
    The key {'month_ID', 'night_length_greek', 'meteo_statement', 'day', 'text_passage', 'day_length_footnote', 'night_length_footnote', 'column', 'addition_text_string_greek', 'addition_text_string', 'night_length', 'type', 'day_length', 'zodiac_part', 'fragment', 'night_length_fractions', 'season', 'month', 'day_length_greek', 'season_ID', 'season_greek', 'day_length_fractions', 'meteo_addition_text_string_greek', 'meteo_addition_text_string', 'length_month', 'zodiac_part_ID'} is not your data, please remove it from the cite!
    Hibeh.json
    5
    You did not commented the key {'meteo_event_class', 'feast', 'feast_greek', 'meteo_event_class_ID', 'feast_ID'} in your cite! Please to so!
    The key {'text_string', 'supplement_Meteo', 'supplement_ID'} is not your data, please remove it from the cite!
    Madrid.json
    6
    You did not commented the key {'addition_ID', 'supplement_ID', 'addition_greek', 'authority', 'status', 'Authority_ID', 'addition'} in your cite! Please to so!
    The key {'feast_ID', 'feast_greek', 'feast'} is not your data, please remove it from the cite!
    Antiochos.json
    7
    You did not commented the key {'authority_ID', 'record_ID', 'parallel_ID', 'parallel'} in your cite! Please to so!
    The key {'addition_ID', 'supplement_ID', 'addition_greek', 'status', 'ID', 'Authority_ID', 'addition'} is not your data, please remove it from the cite!
    

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




    [{'mimetype': 'application/json',
      'updated': '2020-10-14T14:47:30.838604+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Antiochos.json',
       'version': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Antiochos.json?versionId=6d0fb48e-5af2-4a83-a130-7bbfb81afc57',
       'uploads': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Antiochos.json?uploads'},
      'is_head': True,
      'created': '2020-10-14T14:47:30.832807+00:00',
      'checksum': 'md5:9f8b8b88c310dc7256f23ae114eecb10',
      'version_id': '6d0fb48e-5af2-4a83-a130-7bbfb81afc57',
      'delete_marker': False,
      'key': 'Antiochos.json',
      'size': 78439},
     {'mimetype': 'application/json',
      'updated': '2020-10-14T14:47:31.132815+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Geminos.json',
       'version': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Geminos.json?versionId=9120551b-d93f-415f-a116-90676f0cc84e',
       'uploads': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Geminos.json?uploads'},
      'is_head': True,
      'created': '2020-10-14T14:47:31.125574+00:00',
      'checksum': 'md5:84278864159cdef63ed7f6e4dca997a5',
      'version_id': '9120551b-d93f-415f-a116-90676f0cc84e',
      'delete_marker': False,
      'key': 'Geminos.json',
      'size': 303197},
     {'mimetype': 'application/json',
      'updated': '2020-10-14T14:47:31.413782+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Hibeh.json',
       'version': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Hibeh.json?versionId=2629e429-99d9-470d-841d-879228845908',
       'uploads': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Hibeh.json?uploads'},
      'is_head': True,
      'created': '2020-10-14T14:47:31.407519+00:00',
      'checksum': 'md5:8e0d25ec5398bf288250ed6892ba6615',
      'version_id': '2629e429-99d9-470d-841d-879228845908',
      'delete_marker': False,
      'key': 'Hibeh.json',
      'size': 46200},
     {'mimetype': 'application/json',
      'updated': '2020-10-14T14:47:31.695866+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Madrid.json',
       'version': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Madrid.json?versionId=11972ddd-230d-4943-a687-27b6ff0a777c',
       'uploads': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Madrid.json?uploads'},
      'is_head': True,
      'created': '2020-10-14T14:47:31.689574+00:00',
      'checksum': 'md5:7c700e14d8d04d0041db949db451fb9b',
      'version_id': '11972ddd-230d-4943-a687-27b6ff0a777c',
      'delete_marker': False,
      'key': 'Madrid.json',
      'size': 83574},
     {'mimetype': 'application/json',
      'updated': '2020-10-14T14:47:32.197854+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Milet.json',
       'version': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Milet.json?versionId=a4b149cd-c36e-4d86-92b1-0f15a99ef6eb',
       'uploads': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Milet.json?uploads'},
      'is_head': True,
      'created': '2020-10-14T14:47:32.192064+00:00',
      'checksum': 'md5:b1ecb72148b8a5c3e878f3d074b8ad97',
      'version_id': 'a4b149cd-c36e-4d86-92b1-0f15a99ef6eb',
      'delete_marker': False,
      'key': 'Milet.json',
      'size': 28419},
     {'mimetype': 'application/json',
      'updated': '2020-10-14T14:47:32.505985+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Oxford.json',
       'version': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Oxford.json?versionId=4cdd7ed1-68c1-4d78-b3fe-106259db99ab',
       'uploads': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Oxford.json?uploads'},
      'is_head': True,
      'created': '2020-10-14T14:47:32.498672+00:00',
      'checksum': 'md5:e69ecabd1b3049fa858178cd0a4bfe67',
      'version_id': '4cdd7ed1-68c1-4d78-b3fe-106259db99ab',
      'delete_marker': False,
      'key': 'Oxford.json',
      'size': 33330},
     {'mimetype': 'application/octet-stream',
      'updated': '2020-10-14T14:47:32.773611+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Parapegmata.cite',
       'version': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Parapegmata.cite?versionId=4ae782dd-45ad-40ba-858b-72a30cd8e276',
       'uploads': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Parapegmata.cite?uploads'},
      'is_head': True,
      'created': '2020-10-14T14:47:32.767986+00:00',
      'checksum': 'md5:ffe53c1cefcede6529615ca3ca6d7db9',
      'version_id': '4ae782dd-45ad-40ba-858b-72a30cd8e276',
      'delete_marker': False,
      'key': 'Parapegmata.cite',
      'size': 16270},
     {'mimetype': 'application/json',
      'updated': '2020-10-14T14:47:33.025750+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Paris.json',
       'version': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Paris.json?versionId=b982d907-a3b0-472c-bf9c-130965cc566d',
       'uploads': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Paris.json?uploads'},
      'is_head': True,
      'created': '2020-10-14T14:47:33.020506+00:00',
      'checksum': 'md5:6917abbae53d2669ea1e773cf272c0a4',
      'version_id': 'b982d907-a3b0-472c-bf9c-130965cc566d',
      'delete_marker': False,
      'key': 'Paris.json',
      'size': 127034},
     {'mimetype': 'application/json',
      'updated': '2020-10-14T14:47:33.387104+00:00',
      'links': {'self': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Phaseis.json',
       'version': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Phaseis.json?versionId=b6fc6f3b-a8ca-4235-9d7f-d6bc7fd7145e',
       'uploads': 'https://sandbox.zenodo.org/api/files/125bd150-85df-4c93-9ca5-d8ca03b926e7/Phaseis.json?uploads'},
      'is_head': True,
      'created': '2020-10-14T14:47:33.379073+00:00',
      'checksum': 'md5:33deed0fdb34e2ff81a5904430c7d1b1',
      'version_id': 'b6fc6f3b-a8ca-4235-9d7f-d6bc7fd7145e',
      'delete_marker': False,
      'key': 'Phaseis.json',
      'size': 920426}]


