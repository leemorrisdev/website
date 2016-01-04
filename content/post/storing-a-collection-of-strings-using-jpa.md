+++
date = "2015-10-19T20:55:03Z"
tags = ["Development"]
title = "Storing a collection of strings using JPA"

+++

I had a requirement which required me to store a collection of IP addresses against a given entity.  On the face of it, this looks like a job for @OneToMany but the IP’s internally are just stored as Strings, and it felt wasteful to wrap this string up in an object with an ID just for the database.

A quick google on this problem suggested I use @ElementCollection with @CollectionTable to handle the one-to-many mapping for me.  I did this but immediately noticed the generated table didn’t have a primary key, which according to the documentation is not supported.  This smells a little for me, as well as the fact I’m going to have to JOIN that table just to retrieve a few strings.

Rather than using these annotations, I decided on using a JPA AttributeConverter instead.  An attribute converter can be used to convert from one type to another and can be attached to fields as required.  Internally, I want to represent the IP addresses as a Set<String>, and decided I could simply convert this to a delimited list comprising of a single field in the database.  This saves me an extra JOIN statement, and also removes my primary key issue.

My converter class is as follows:

    @Converter
    public class StringSetConverter implements AttributeConverter<Set, String> {
 
        private static final String SEPARATOR = ",";
 
        @Override
        public String convertToDatabaseColumn(Set attribute) {
 
            if(attribute == null) {
                return "";
            }
 
            StringBuilder sb = new StringBuilder();
 
            Iterator<?> itemIterator = attribute.iterator();
 
            while(itemIterator.hasNext()) {
                sb.append(itemIterator.next());
                if(itemIterator.hasNext()) {
                    sb.append(SEPARATOR);
                }
            }
 
            return sb.toString();
        }
 
        @Override
        public Set convertToEntityAttribute(String dbData) {
            if(StringUtils.isEmptyOrWhitespaceOnly(dbData)) {
                return Sets.newHashSet();
            }
            return Sets.newHashSet(dbData.split(SEPARATOR));
        }
    }

To assign the converter, you need to add the following to your field:

    @Column
    @Convert(converter = StringSetConverter.class)
    private Set allowedSourceIps;

Now, I can add my IP addresses to the Set and they’ll be represented as a pipe delimited String in my column:

    Sets.newHashSet("192.168.1.2", "192.168.1.3") ---> '192.168.1.2,192.168.1.3'

