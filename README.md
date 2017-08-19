## Lifion Case Study

#### Data Modeling
> Given the above requirements please create a data model in the form of a complete physical entity relationship diagram (ERD). Your diagram should be comprehensive and sufficient for creating a relational database schema that will support the applications’ requirements.

The ERD can be downloaded in PDF format from [here](https://www.lucidchart.com/publicSegments/view/f076239c-31f1-465d-aeb1-07cae297c933).

#### Query
> Based on your datamodel please compose a query listing all Rental Vehicles which are available on the indicated Start and End Dates.

The following query returns the vehicles that are available on the specified beginning and end dates.

```SQL
SELECT * FROM vehicles
LEFT JOIN booking_dates
ON vehicles.id = booking_dates.vehicle_id
WHERE date IS NULL
OR date <> (
  SELECT start_date
  FROM reservations
  WHERE id = 1
)
AND date <> (
  SELECT end_date
  FROM reservations
  WHERE id = 1
);
```

The above query does not, however, account for the vehicle being unavailable for dates between the start and end date. Therefore the following query is recommended:

```SQL
SELECT * FROM vehicles
LEFT JOIN booking_dates
ON vehicles.id = booking_dates.vehicle_id
WHERE date IS NULL
OR date NOT BETWEEN (
  SELECT start_date
  FROM reservations
  WHERE id = 1
)
AND (
  SELECT end_date
  FROM reservations
  WHERE id = 1
);
```
A fiddle can be found [here](http://sqlfiddle.com/#!9/656a83/6).

#### Algorithm

The Algorithm is written in Ruby and begins with a call to the `top_n_models_for_cat` method. This method makes an inital call to the database to retrieve an array of `Category` records/objects.

```SQL
SELECT * FROM categories;
```

This is then represented by `Category.all`.

```ruby
#[<Category:0x00000004d07a60 id: 2, name: "truck">
#<Category:0x00000004d078f8 id: 1, name: "car"]
```

Additionally, the top `n` modles can be specified as the first argument in the form of an integer.

For every category, the name attribute is used as a key in a `data` hash. This key points to the sorted array of top `n` models.

Note: The following code assumes that each `Category` object has a `models` method which returns an array of `Model` objects.

```ruby
def top_n_models_for_cat(n)
  data = {}

  Category.all.each do |cat|
    data[cat.name] = sort_models(cat.models).first(n)
  end

  return data
end
```

The main sorting algorithm begins in the `sort_models` method, taking in the array of models. It then implements a merge sort, first breaking the array down into single element arrays and the calling `merge_models` to merge the arrays, greatest to least.

```ruby
def merge_sort(array)
  return array if array.length == 1

  mid = array.length / 2

  sub_left = array.slice(0, mid)
  sub_right = array.slice(mid, array.length - mid)

  sorted_sub_left = merge_sort(sub_left)
  sorted_sub_right = merge_sort(sub_right)
 

  return merge_models(sorted_sub_left, sorted_sub_right)
end

def merge_models(a, b)
  result = []

  while !a.empty? && !b.empty?
    if a.first.days_booked > b.first.days_booked
      result << a.shift
    else
      result << b.shift  
    end  
  end

  remaining = a.empty? ? b : a
  return result + remaining
end
```

A ruby fiddle can be found [here](https://repl.it/KQQH/0).