with own as -- Имя страны Россия указано
(
	select 
    	u.id,
    	case 
    		when true then 'russia' end as rus
	from
    	case10.users u 
	join
    	case10.addresses a
        	on 
            	u.id=a.addressable_id
	join 
    	case10.cities c
        	on
            	a.city_id=c.id
	join 
    	case10.regions r
        	on
            	c.region_id=r.id
	join 
    	case10.countries co
        	on
            	r.country_id=co.id
	where co.name = 'Russia'
),nomb as  -- Определение России по телефонному номеру
(
	select
		u.id,
		phone,
		case
			when left(u.phone, 2) ~ '^(79|78|73|74).*' and length(u.phone) = 11 then 'russia'
			when u.phone is null then null
			else 'non-russia'
		end as is_russia
	from
		gd2.users u
), two as -- Преобразования 
(		
	select 
		n.id,
    	case 
    		when true then 'russia' end as rus
	from 
		nomb n
	where
		n.is_russia = 'russia'
),ip as -- Определение России по ip
(
		select 
			(u.last_sign_in_ip::inet - '0.0.0.0'::inet)::bigint decimals,
			u.id
		from 
			case10.users u 	
),ip_ru as -- ip России
(
		select *
		from 
			case10.ip2location_db1 ild 
		where ild.country_name in ('Russian Federation') 
),three as -- Преобразование в удобный список
(
	select 
		i.id,
    	case 
    		when true then 'russia' end as rus	
	from 
		ip i
			join
				ip_ru ir
				on i.decimals >= ir.ip_from and i.decimals <= ir.ip_to
),russian_id as -- комбинация списков
(
select 
	u2.id
from 
	case10.users u2 
		left join 
			own o
			on u2.id = o.id
				left join
					two t
					on u2.id = t.id
						left join
							three th
							on u2.id = th.id
where o.rus = 'russia' or t.rus = 'russia'or th.rus = 'russia'
),registration_russia_per_month as -- первая часть конечной таблицы
(
select 
    date_trunc('month',u.created_at) registration_month,
    count(distinct u.id) quantity
from
    case10.users u
where
	exists(
		select *
        from russian_id
        where russian_id.id = u.id)		
group by 1
),cohorts_russia as -- окончательный вид exists отфильтровывает пользователей из Росси, можно использовать not для отражения
(
select 
	date_trunc('month', u.created_at) as registration_month,
	reg.quantity as registrated_users,
    date_trunc('month', buy.purchased_at) as purchase_month,
    count(distinct buy.id) as purchase_quantity,
    count(distinct buy.id)*1.0 / reg.quantity as conversion_rate
from 
	case10.users u
	join 
		registration_russia_per_month reg
		on reg.registration_month = date_trunc('month', u.created_at)
			join
				case10.carts buy 
				on buy.user_id = u.id and buy.state = 'successful'
where
	exists(
		select *
        from russian_id
        where russian_id.id = u.id)
group by 1, 2, 3
order by 1, 3
)
select -- Запрос для выгрузки
    *
from
    cohorts_russia