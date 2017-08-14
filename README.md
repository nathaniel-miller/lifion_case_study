## Lifion Case Study
Note: The following answers assume the use of PostgresQL

#### Data Modeling
> Given the above requirements please create a data model in the form of a complete physical entity relationship diagram (ERD). Your diagram should be comprehensive and sufficient for creating a relational database schema that will support the applications’ requirements.

The ERD can be downloaded in PDF format from [here](https://www.lucidchart.com/publicSegments/view/f076239c-31f1-465d-aeb1-07cae297c933).

#### Query
> Based on your datamodel please compose a query listing all Rental Vehicles which are available on the indicated Start and End Dates.

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
A fiddle can be found [here](http://sqlfiddle.com/#!17/21d3a/2).

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
get_top_models(5, rentals_hash)
```

Note: The following code assumes an initial query to the database for all categories. This is represented as `Category.all`. It is also assumed that each `Category` object has a `vehicles` method which returns an array of `Vehicle` objects.

The `top_n_rentals`, `consolidate_bookings`, and `vehicle_hash_setup` methods are used for generating a properly structured hash as indicated above. 

```ruby
def top_n_rentals(n)
  cat_hash = consolidate_bookings(Category.all)
  
  return get_top_models(cat_hash, n)
end

def consolidate_bookings(categories)   
  all_bookings = {}

  categories.each do |cat|
    cat_bookings = vehicle_hash_setup()

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

The main sorting/ranking algorithm begins by taking the structured data and iterating through the data with the `manage_` functions.

```ruby
def get_top_models(n = 5, cat_hash)

  data = {
    :sorted => {},
    :cat_hash => cat_hash
  }

  return manage_categories(data, n)[:sorted] 
end  

def manage_categories(data, n)
  
  data[:cat_hash].each do |cat, make_hash|
    data[:sorted][cat] = []
    data[:current_cat] = cat
    data[:current_make_hash] = make_hash

    data = manage_makes(data)
    data[:sorted][cat] = data[:sorted][cat].first(n)
  end
  
  return data
end

def manage_makes(data)
  data[:current_make_hash].each do |make, model_hash|
    data[:current_make] = make
    data[:current_model_hash] = model_hash

    data = manage_models(data) 
  end

  return data
end

def manage_models(data)
  data[:current_model_hash].each do |model, bookings|

    data[:current_model] = model
    data[:current_bookings] = bookings

    data = pre_sort_data(data)
  end

  return data
end
```

The heart of the actual algorithm resides in the `pre_sort_data`, `sort_extreme`, and `sort_midrange` functions.

```ruby
def pre_sort_data(data)
  sorted_cats = data[:sorted][data[:current_cat]]

  if sorted_cats.empty?
    sorted_cats << bookings_hash(data)
  else
    data = sort_extreme(data)
  end

  return data
end

def sort_extreme(data)
  sorted_cats = data[:sorted][data[:current_cat]]
  booking_count = data[:current_bookings]
  booking_data = bookings_hash(data)


  most_bookings  = extract_booking_count(sorted_cats.first)
  least_bookings = extract_booking_count(sorted_cats.last)

  if booking_count >= most_bookings
    sorted_cats.unshift booking_data
  elsif booking_count < least_bookings
    sorted_cats << booking_data
  else
    sort_midrange(booking_count, booking_data, data)
  end

  return data
end

def sort_midrange(booking_count, booking_data, data)
  sorted_cats = data[:sorted][data[:current_cat]]

  l = 0
  h = sorted_cats.length - 1

  while true do
    i = (l + h) / 2
    mid_count = extract_booking_count(sorted_cats[i])

    if booking_count >= mid_count
      i > 0 ? index = i : index = 0
      more_than_mid = extract_booking_count(booking_data[index])

      if more_than_mid <= booking_count
        sorted_cats.insert(i, booking_data)
        break
      else
        h = i
      end

    else
      i < sorted_cats.length - 1 ? index = i + 1 : index = i
      less_than_mid = extract_booking_count(sorted_cats[index])

      if less_than_mid < booking_count
        sorted_cats.insert(i + 1, booking_data)
        break
      else
        l = i
      end

    end
  end
end

def bookings_hash(data)
  {
    data[:current_make] => {
      data[:current_model] => data[:current_bookings]
    }
  }
end  

def extract_booking_count(hash)
  # expects hash => {"make" => {"model" => n}}
  make = hash.keys[0]
  model = hash[make].keys[0]

  hash[make][model]
end
```
A call to `get_top_modles` with data structured in the manner below will result in an hash of categories, each pointed to an array sorted from most popular to least popular. 

```ruby
puts get_top_models(5,{
  'car' => {
    "ford"=>{
      "focus"=>30
    }, 
    "ferrari"=>{
      "testarossa"=>9
      }
    },
  'truck' => {
    "ford"=>{
      "ranger"=>30, 
      "monster truck"=>12, 
      "F-150" => 20, 
      "edge" => 2, 
      "explorer" => 32,
      "expedition" => 14
    }, 
    "dodge"=>{
      "ram"=>9
      }
    },
  'van' => {
    "uhaul" => {
      "moving van" => 101
    }
  }  
})
```
```ruby
{
  "car"=>[
    {"ford"=>{"focus"=>30}}, 
    {"ferrari"=>{"testarossa"=>9}}
  ], 
  "truck"=>[
    {"ford"=>{"explorer"=>32}}, 
    {"ford"=>{"ranger"=>30}}, 
    {"ford"=>{"F-150"=>20}}, 
    {"ford"=>{"expedition"=>14}}, 
    {"ford"=>{"monster truck"=>12}}
  ], 
  "van"=>[
    {"uhaul"=>{"moving van"=>101}}
  ]
}
```

A ruby fiddle can be found [here](http://rubyfiddle.com/riddles/26441).







