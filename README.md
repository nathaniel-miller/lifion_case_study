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
A fiddle can be found [here](http://sqlfiddle.com/#!17/21d3a/1).

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
find_top(5, hash)
```

Note: The following code assumes an initial query to the database for all categories. This is represented as `Category.all`. It is also assumed that each `Category` object has a `vehicles` method which returns an array of `Vehicle` objects.

The `top_n_rentals`, `consolidate_bookings`, and `vehicle_hash_setup` methods are used for generating a properly structured hash as indicated above. The main sorting/ranking algorithm is found in the `find_top` method.

```ruby
def top_n_rentals(n=5)
  models_in_categories = consolidate_bookings(Category.all)


  return find_top(n, models_in_categories)

end

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

def find_top(n = 5, cats_hash)
  hash = {}

  cats_hash.each do |cat, make_hash|
    hash[cat] = []

    make_hash.each do |make, model_hash|
      model_hash.each do |model, bookings|

        if hash[cat].empty?
          hash[cat] << {make => {model => bookings}}
        else
          l_index = 0
          h_index = hash[cat].length - 1

          while true do
            highest_count  = extract_booking_count(hash[cat].first)
            lowest_count   = extract_booking_count(hash[cat].last)
            
            if bookings >= highest_count
              hash[cat].unshift({make => {model => bookings}})
              break
            elsif bookings < lowest_count
              hash[cat].push({make => {model => bookings}})
              break
            else
              i = (l_index + h_index) / 2
              mid_count = extract_booking_count(hash[cat][i])

              if bookings >= mid_count
                i > 0 ? index = i : index = 0
                adjacent_more = extract_booking_count(hash[cat][index])

                if adjacent_more <= bookings
                  hash[cat].insert(i, {make => {model => bookings}})
                  break
                else
                  h_index = i
                end

              else
                i < hash[cat].length - 1 ? index = i + 1 : index = i
                adjacent_less = extract_booking_count(hash[cat][index])

                if adjacent_less < bookings
                  hash[cat].insert(i + 1, {make => {model => bookings}})
                  break
                else
                  l_index = i
                end
              end 
            end      
          end
        end  
      end
    end
    
    hash[cat] = hash[cat].first(n)
  end 
  
  return hash
end

def extract_booking_count(hash)
  # expects hash => {"make" => {"model" => n}}
  make = hash.keys[0]
  model = hash[make].keys[0]

  hash[make][model]
end 
```

A ruby fiddle can be found [here](http://rubyfiddle.com/riddles/26441).







