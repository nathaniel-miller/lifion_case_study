### Lifion Case Study
Note: The following answers assume the use of PostgresQL

#### Data Modeling
The ERD can be downloaded in PDF format from:
[https://www.lucidchart.com/publicSegments/view/5063a6c5-b8af-4f4c-8fc9-b3dc94cf984c](https://www.lucidchart.com/publicSegments/view/5063a6c5-b8af-4f4c-8fc9-b3dc94cf984c)

#### Query
Based on your datamodel please compose a query listing all Rental Vehicles which are available on the indicated Start and End Dates.

The following query returns the vehicles that are available on the specified beginning and end dates.

```SQL
SELECT * FROM vehicles
WHERE NOT '2017-08-01' IN ('{dates_booked}')
AND NOT '2017-08-28' IN ('{dates_booked}');
```

The above query does not, however, account for the vehicle being unavailable for dates between the start and end date. Therefore the following query is recommended:

```SQL
SELECT * FROM vehicles
WHERE NOT daterange('2017-08-12', '2017-08-14')  @> ANY (dates_booked);
```
A fiddle can be found here:(http://sqlfiddle.com/#!17/21d3a/1)[http://sqlfiddle.com/#!17/21d3a/1]

#### Algorithm

The Algorithm is written in Ruby and assumes a hash as input in the following format:
```ruby
{
  'category_1' => {
    'make_1' => {
      'model_1' => int,
      'model_2' => int
    },
    'make_2' => {
      'model_1' => int,
      'model_2' => int      
    }
  },
  'category_2' => {
    'make_1' => {
      'model_1' => int,
      'model_2' => int
    },
    'make_2' => {
      'model_1' => int,
      'model_2' => int      
    }    
  }
}
```

Additionally, the top `n` can be specified as the first argument in the form of an integer.

```ruby
top(5, hash)
```

Note: The submitted algorithm is probably not reflective of the way I would actually produce a list of 'Top 5 Makes/Models' in each category.

In reality, I would add a `rank` column to the `vehicles` table, update the `rank` column with the following code:

```ruby
#assume categories is a queried array of category objects/records 
#each with a vehicles method that retrieves an array of vehicles

def consolidate_bookings(categories)
    
  all_bookings = {}

  categories.each do |cat|

    cat_bookings = vehicle_hash_setup

    cat.vehicles.each do |v|
      cat_bookings[v.make][v.model] += v.dates_booked.count
      
    end

    all_bookings[cat.name] = cat_bookings
    
  end

  all_bookings

end

def vehicle_hash_setup

  Hash.new do |make_hash, make|
    make_hash[make] = Hash.new do |model_hash, model|
      model_hash[model] = 0
    end  
  end

end
```







