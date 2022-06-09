---
title: "Visualising some school data because I had nothing better to do"
date: 2022-03-27T08:19:11+05:30
draft: false
---

So last night, I had the "brilliant" idea of archiving, my school's weekly
schedule for the session of 2021-2022, this was possible due to the [unofficial
API wrapper](https://github.com/Aadv1k/schoolOS-api-wrapper) I made for
schoolOS, the site which my school uses to schedule assignments and classes, but
why? why would I archive this data? well for two reasons first, since this
session is gonna end the schedule and the data for class 9th under my ID is
going to be revoked and the second reason being, I had nothing better to do.

So work was made a lot easier since I already had a wrapper, all I needed was to
somehow figure out the dates, the dates required by `get_timetable(start, end)`
has to be provided in a particular way of being sunday to saturday and in the
format of `yyyy-mm-dd`. This was easy to figure out with some quick math.

```python
def gen_months(months, year, timetable):
    for mdate,mname in enumerate(months, start=1):
        m = f'{mdate}_{mname}_{year}'
        os.mkdir(m)
        d = 1

        for i in range(4):
            with open(f'{m}/{i}.json', 'w') as file:
                file.write(json.dumps(timetable(f"2021-{mdate}-{d}", f"2021-{mdate}-{d+6}")))
            d+=1+6
```

this code can be found at [examples/gen_date_for_months](https://github.com/Aadv1k/schoolOS-api-wrapper/blob/bfe41cd23c617625331186bb2d7ab3bd3c4b0162/examples/gen_date_for_months.py) in the schoolOS-api-wrapper.

basically, we pass in a list of months, we loop through an enumeration of the
list, and for each month, we loop four times, each time we request a new
timetable with the current month, and the "date", which is initially 1, but then
we simply add 7 to it. This work surprisingly well, giving us a clean data like so

```
├── 10_oct_2021/
│   ├── 0.json
│   ├── 1.json
│   ├── 2.json
│   └── 3.json
├── 11_nov_2021/
├── 12_dec_2021/
├── 4_apr_2021/
├── 5_may_2021/
├── 6_jun_2021/
├── 7_jul_2021/
├── 8_aug_2021/
└── 9_sep_2021/
```

So once I had the data, it was now time to actually figure it out; The data
structure was like so `data['data']['groups']` the `groups` is an array of all
the days of the week that have at least one class. this group itself, is in the
following structure --

```json
{
    "label": "Monday",
    "date": "05 Apr, 2021",
    "groups": [...]
}
```

Where `groups` yet again is an array containing all the classes of the day. Each
class gives us quite a lot of data to work with, providing us with the `id`,
`title`, `start_time`, `end_time`, `has_classwork`, `teacher: {}`, `attendance`

`has_classwork` is a boolean value which is set to true if the teacher decided
to decided to conduct some classwork then actually log it in the portal. This
gave me an idea, now that I had the cleaned up date, I could do quite a lot of
stuff with it, namely, visualizing which subject had the most classworks, so
here is How I did it.

```python
import glob
import json
from datetime import datetime, date
import matplotlib.pyplot as plt

file_list = glob.glob('./*/**/*.json', recursive=True)
data = [json.load(open(file)) for file in file_list]
filter(lambda x: len(x['data']['group']) != 0, data)

final_data = []

for i in data:
    for j in i['data']['groups']:
        for o in j['groups']:
            for l in o['lectures']:
                cur = {}

                cur['title'] = l['title']
                cur['has_classwork'] = l['has_classwork']
                cur['teacher_name'] = l['teacher']['name']
                cur['attendance'] = l['attendance']

                if (l.get('lecture_media')):
                    if(l['lecture_media'].get('presentations')):
                        cur['presentation_url'] = l['lecture_media']['presentations'][0]['content']['attachment']['media_url']

                    if(l['lecture_media'].get('videos')):
                        cur['video_url'] = l['lecture_media']['videos'][0]['content']['youtube_url']
                final_data.append(cur)


class_work_data = {}

for classes in final_data:
    tname = classes['title']
    if classes['has_classwork']:
        if tname in class_work_data:
           class_work_data[tname] += 1
        else:
           class_work_data[tname] = 0
           class_work_data[tname] += 1

data = list(class_work_data.values())
labels = ['\n'.join(names.split(' ')) for names in class_work_data.keys()]

plt.xticks(range(len(data)), labels)
plt.xlabel('Subject')
plt.ylabel('Total classwork')
plt.title('classwork data 2021-2022')

plt.bar(range(len(data)), data, color='royalblue', alpha=0.7)
plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
plt.show()
```

this file is contained in the directory where all the data is placed, use the
`glob.glob` to get the names of all json files, we then loop through all the
classes in all the days, in all the files to fetch the data we need, I put in
more than just the `has_classwork` and included other data for future use in the
`final_data` dict. The final result was quite interesting.

![Classwork-data-plot-img](/classwork-plot-img.png) 

The reason why the data has huge disparities between them, can be credited to
two reasons 
- Not all classwork is put in as "Classwork" in the portal
- Most of the time in place of a "Classwork" external sites such as quizzz or
  kahoot were used in place of a classwork.

