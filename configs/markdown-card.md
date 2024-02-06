Time &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Price
{%- set results = state_attr('sensor.octopus_agile_prices', 'array') %}
{%- for record in results %}
{%- set ts = as_timestamp(record.from) %}
{%- set ts_now = ((as_timestamp(now())/1800)|round(0,'floor')|int * 1800) %}
{%- if ts >= ts_now %}
{{ ts|timestamp_custom ('%H:%M', local=true)
}}&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; {{
record.export }}
{%- endif %}
{%- endfor %}
