```python
# -*- coding: utf-8 -*-
# ---
# jupyter:
#   jupytext:
#     formats: ipynb,py
#     text_representation:
#       extension: .py
#       format_name: light
#       format_version: '1.5'
#       jupytext_version: 1.4.2
#   kernelspec:
#     display_name: Python 3
#     language: python
#     name: python3
# ---

from gssutils import *
from gssutils.metadata import THEME
scraper = Scraper('https://www.ons.gov.uk/economy/grossdomesticproductgdp/datasets/realtimedatabaseforukgdpybha')
scraper

# +
dist = scraper.distributions[0]
tabs = (t for t in dist.as_databaker())
tidied_sheets = []

def left(s, amount):
    return s[:amount]
def right(s, amount):
    return s[-amount:]

def date_time(time_value):
    time_string = str(time_value).replace(".0", "").strip()
    time_len = len(time_string)
    if time_len == 4:
        return "year/" + time_string
    elif time_len == 7:
        return "quarter/{}-{}".format(time_string[3:7], time_string[:2])
    elif time_len == 10:       
        return 'gregorian-interval/' + time_string[:7] + '-01T00:00:00/P3M'




# +

trace = TransformTrace()

for tab in tabs:
        if (tab.name == '1989 - 1999') or (tab.name == '2000 - 2010') or (tab.name == '2011 - 2017') or (tab.name == '2018 -'):
            
            datacube_name = "GDP at current prices – real-time database (YBHA)"
            columns=["GDP Reference Period","Publication Date","GDP Estimate Type","Measure Type"]
            trace.start(datacube_name, tab, columns, scraper.distributions[0].downloadURL)
            
            trace.GDP_Reference_Period("Selected as non blank values below and including cell A6")
            vintage = tab.excel_ref('A6').expand(DOWN).is_not_blank()        
                
            estimate_row = 6
            publication_row = 7
            if (tab.name == '2011 - 2017') or (tab.name == '2018 -'):
                estimate_row = 5
                publication_row = 6
            
            trace.GDP_Estimate_Type("Selected as non blank values to the right and inlcuding cell B{}", var=estimate_row)
            estimate_type = tab.excel_ref('B{}'.format(estimate_row)).expand(RIGHT).is_not_blank()
                
            trace.Publication_Date("Selected as non blank values to the right and inlcuding cell B{}", var=publication_row)
            publication = tab.excel_ref('B{}'.format(publication_row)).expand(RIGHT).is_not_blank()
        
            trace.Measure_Type("Hard coded to {}", var="GBP Million")
            
            trace.OBS("Any non blank values - below - any non blank cells to the right and inlcuding cell B{}".format(publication_row))
            observations = publication.fill(DOWN).is_not_blank()
        
            dimensions = [
                HDim(vintage, trace.GDP_Reference_Period.label, DIRECTLY, LEFT),
                HDim(estimate_type, trace.GDP_Estimate_Type.label, DIRECTLY, ABOVE),
                HDim(publication, trace.Publication_Date.label, DIRECTLY, ABOVE),
                HDimConst(trace.Measure_Type.label, trace.Measure_Type.var),
            ]

            tidy_sheet = ConversionSegment(tab, dimensions, observations)
            trace.with_preview(tidy_sheet)
            
            trace.store("combined_dataframe", tidy_sheet.topandas())


# +
df = trace.combine_and_trace(datacube_name, "combined_dataframe")

trace.add_column("Marker")
trace.add_column("Value")
trace.multi(["Marker", "Value"], "Rename databaker columns OBS and DATAMARKER columns to Value and Marker respectively")
df.rename(columns={'OBS' : 'Value','DATAMARKER' : 'Marker'}, inplace=True)

trace.Publication_Date("Remove whitespace between the Quarter and Year values")
df['Publication Date'].replace('Q3  1990', 'Q3 1990', inplace=True)  #removing space
df['Publication Date'].replace('Q 2004', 'Q4 2004', inplace=True) #fixing typo

trace.GDP_Reference_Period("Replace typo'd year 1010 with 2010")
df['GDP Reference Period'].replace('Q2 1010', 'Q2 2010', inplace=True) #fixing typo

trace.multi(["GDP_Reference_Period", "Publication_Date"], "Addly function to reformat time")
df["GDP Reference Period"] = df["GDP Reference Period"].apply(date_time)
df["Publication Date"] = df["Publication Date"].apply(date_time)

trace.Marker("Replace '..' marker with 'unknown'.")
df['Marker'].replace('..', 'unknown', inplace=True)
# -

tidy = df[['GDP Reference Period','Publication Date','Value','GDP Estimate Type', 'Measure Type','Marker']]
trace.GDP_Estimate_Type("Remove any prefixed whitespace from all values in column.")
for column in tidy:
    if column in ('GDP Estimate Type'):
        tidy[column] = tidy[column].str.lstrip()
        tidy[column] = tidy[column].str.rstrip()
tidy

out = Path('out')
out.mkdir(exist_ok=True)
trace.ALL("Remove all duplicate rows from dataframe.")
tidy.drop_duplicates().to_csv(out / 'observations.csv', index = False)

# +
scraper.dataset.family = 'trade'

## Adding short metadata to description
additional_metadata = """ All data is seasonally adjusted.
In July 2018, a new GDP publication model was adopted. Quarterly estimates of GDP have since been published twice a quarter, rather than three times a quarter as happened prior to this.
"""
scraper.dataset.description = scraper.dataset.description + additional_metadata

from gssutils.metadata import THEME

with open(out / 'observations.csv-metadata.trig', 'wb') as metadata:
    metadata.write(scraper.generate_trig())

# +
csvw = CSVWMetadata('https://gss-cogs.github.io/family-trade/reference/')
csvw.create(out / 'observations.csv', out / 'observations.csv-schema.json')

trace.output()
```
