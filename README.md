## Unstructured Data - Final Exam/Project

For this project, we are looking at data from an online retailer, where we mainly focus on items that were purchased together.
To do so, we pivot the data and **group by** the **InvoiceNo** column to learn which items were purchased together,
by invoice number.  This therefore can be our * **basket** *. 

From here, we can push this data to ElasticSearch, which will then analyze the data to tell us which items
are often purchased together.  This can help buyers, analysts, managers, and advertisers understand how to 
better manage their proudcts when buying inventory and advertising products programmatically.

First, to decompress the tar.gz file.  Run the below code in the folder you want to run the project to decompress the zipped file:
```
tar xvzf notebook.tar.gz
```

The below outlines the .ipynb file **'Final_Exam'**.

```
# Import any necessary libraries 
import pandas as pd
import requests
import pprint

df = pd.read_csv('Online_Retail.csv')
```

_**Note**: If you do not have the file saved as a csv, you can use the below code to convert the file from a .xlsx file to a .csv.  This code also includes some formatting cleanup of the data as well.

```
df = pd.read_excel("Online Retail.xlsx")
# filter out missing/bad data.  Not sure of the implications here
df = df.dropna()

# for some reason CustomerID is interpreted as float.  Change to int
df['CustomerID'] = df['CustomerID'].astype(int)

df.to_csv("Online_Retail.csv", index=False)
```

## **Group the data by a unique identifier - in this case, the invoice number**
We want to re-create the "basket" that a customer creates based on the *InvoiceNo* column.
This aggregation allows us to see which products were in the same basket and can later helps us understand which products are most often purchased together.

**Here's a quick way to use the groupBy function built into the Pandas library:**
```
invoice_df = df.groupby('InvoiceNo')
```


The next few cells of code create indices and fucntions that we use to fling the data to Elasticsearch.

1. This code creates the function to build the indices we need for Elasticsearch and eventually, Kibana:


2. The below are two functions create to create and delete the indices:
```
def create_es_index(index, index_config):
    r = requests.put("http://elasticsearch:9200/{}".format(index),
                     json=index_config)
    if r.status_code != 200:
        print("Error creating index")
    else:
        print("Index created")
        

def delete_es_index(index):
    r = requests.delete("http://elasticsearch:9200/{}".format(index))
    if r.status_code != 200:
        print("Error deleting index")
    else:
        print("Index deleted")
```


3. Here we delete the index if exists, if not, we create the final index for reference, based on the data and the format of the data.  Additionally, we turn the elasticsearch analyzer off to ensure that the coded does indeed run :
```
# delete if exists
indices, indices_full = get_es_indices()
if 'order' in indices:
    delete_es_index('order')
    
index_config = {
    "mappings": {
        "orders": {  # document TYPE
            "properties": {
                "InvoiceNo": {"type": "string"},
                "invoice": {"type": "string"},
                "Descriptions": {"type":"string", "index":"not_analyzed"}, 
                "StockCode": {"type":"string"}
            }
        }
    }
}

create_es_index('order', index_config)
```

## Next, we get ready to fling the data to elasticsearch.
### First, a function is defined to fling the message to elastic search.
#### Next, a loop is created to iterate over each invoice; the data is then added to a new dataframe and columns within the new dataframe.  We include the following variables:
- InvoiceNo (invoice number)
- CustomerId (the unique customer ID)
- InvoiceDate (the date the invoice was sent)
- Country (the country in which the order was placed)
- StockCode (the unique ID code for the product purchased)
- Description (product name description)
- Quantity (the total quantity of the specific product purchased, by invoice)
- UnitPrice (the price of the proudct at the time of purchase)

## Last, we use the defined function to fling the data to Elasticsearch with the below code:
```
fling_message('order', 'orders', basket)
```

If wanted, one can print the **basket** dataframe to inspect if it collected and appended the data correctly with the below code.  Made sure to comment out one or the other, but it's best to not run both at the same time.
```
print(basket)
```

### When running the above code where the data is "flinged" to elasticsearch, you will need to **interrupt** or **stop** the cell from running after a few seconds.  It will continue to run if you do not stop it.  Once stopped or interrupted, you can continue on to the next block of code.


## Using Lucene, a query language, to aggregate the data to create "basket" recommend products to a customer based on what is currently in their cart.  The recommendation comes from past purchases on similar products from past customers, and products that are highly correlated to those past purchases.


Once the cell with the recommender function is run, you can then pick a product to create a recommendation with: 

```
product = 'SET 7 BABUSHKA NESTING BOXES'
```

Then, you print out the recommendation for *'product'* against the *order* index.

```
res = execute_es_recommender_query('order', product)
pp = pprint.PrettyPrinter(indent=4)
pp.pprint(res['aggregations'])
```

Last, we create a recommended products to purchse list, based on the analysis of the data, for products correlated or recommended to purchse with 'SET 7 BABUSHKA NESTING BOXES':

```
recommended_products = res['aggregations']['highly_correlated_words']['buckets']

for item in recommended_products:
    print(item['key'], ' ', item['score'])
```

Here is the list of items that are recommended to be purchased with the 'SET 7 BABUSHKA NESTING BOXES':
```
GLASS STAR FROSTED T-LIGHT HOLDER   5.572321292485902
WHITE METAL LANTERN   5.540123456790123
SAVE THE PLANET MUG   5.153635116598079
WOODEN PICTURE FRAME WHITE FINISH   5.145496113397347
VINTAGE BILLBOARD LOVE/HATE MUG   5.145496113397347
VINTAGE BILLBOARD DRINK ME MUG   4.791495198902606
RETRO COFFEE MUGS ASSORTED   4.47914145081901
WOOD S/3 CABINET ANT WHITE FINISH   4.402286236854137
CREAM CUPID HEARTS COAT HANGER   4.402286236854137
WOOD 2 DRAWER CABINET WHITE FINISH   4.201493674744703
```
From this analysis, analysts and tell data scientists which products pair best with which.  From there, an algorithm could be built to programmatically advertise or recommend those products to users who have the 'SET 7 BABUSHKA NESTING BOXES' product, or any other product for which the recommendation is built, to purchase as well.  

