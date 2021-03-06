# README
Based on [mod_xml_curl](https://freeswitch.org/confluence/display/FREESWITCH/mod_xml_curl) module.

This is a script for using PHP to generate dynamic XML. My script is based on a simple mySQL class. I know there's a PDO class and if someone wanted to walk me through it for sqlite/postgres compatibility I'd be interested.
* It currently only does the dialplan (while xml_curl can be used for ANY FreeSWITCH XML configuration).
* and only the default context

# BENEFITS:
1. mod_lcr with advanced bill-determining capability. I want to further abstract it to make it easy to do a function call) so you can list the prices for users who search by a number.
2. mod_lcr with an LRN table. If I had a callwithus account, I'd implement LRN lookup, it's cheap enough that it's worth it, and keep the db up to date.
3. easy e.164 number translation without extra transfers and tons of extensions. Just one line of code for each transformation.
4. local enum lookup, easily. And you can even put it after your billing code.
5. ***and maybe my main one*** - drastically customize what goes on in the dial plan based on the user's preferences, wherever they are stored.

# TODO:
1. Abstract my billing algorithm from the mod_lcr recode.
2. restore the default mod_lcr query - I merged things into 2 tables instead of 3 hoping to improve query time, but that was unnecessary.

# Please follow all these steps:
## 1. Setting up XML curl:
XML is located in `freeswitch/src/mod/xml_int/mod_xml_curl` and you'll have to
   
1. make && install it, if you hadn't done so already.
2. Add it to your modules.conf for auto load, `<load module="mod_xml_curl"/>`.


## 2. XML curl config files:
You will need to edit/create a xml_curl.conf setup. If you are doing live-editing at all, php is prone to parse errors. So set up a backup of your old "safe" file just incase you save incorrectly.
Or, you can really do redundancy and set it to use another host. This is set to only be the dialplan.

```xml
<configuration name="xml_curl.conf" description="cURL XML Gateway">
  <bindings>
    <binding name="dialplan fetcher">
      <param name="gateway-url" value="http://localhost/xml.php" bindings="dialplan"/>
    </binding>
    <binding name="backup dialplan fetcher">
      <param name="gateway-url" value="http://localhost/xml_backup.php" bindings="dialplan"/>
    </binding>
  </bindings>
</configuration>
```

Then, `load mod_xml_curl` in the fscli or `reload mod_xml_curl` if you hadn't done so yet.


## 3. Dependencies in PHP
This relies on the XMLWriter class in php to make things easy. It probably wasn't necessary - the XML isn't THAT complicated, but that's the example from the wiki and it did make things easy.
Anyway, if you don't have it, this won't work. Google it and compile the extension.

## 4. My code makes various DB calls, especially for the mod_lcr. If you don't need this, remove any $db-> references.
Update the bottom of db.php file with your DB access.
I think having GLOBAL $db in the functions is a little cheesy, but I don't really know a good alternative.
Also, if you are using a DB other than mysql, when calling "expand_digits(" function, you can delete or set the 2nd parameter to 0, probably.

## 5. FreeSWITCH relies on the dynamic XML and won't look at the static XML at all, unless you send a `<result status="not found" />`
This line can be anywhere inside the proper XML, it seems.
Since I start a context w/o knowing if I actually have it in the dynamic XML, I have a little variable - when I've *actually* matched the call, set $do++;
At the end, if $do==0, then it must be I haven't had a result, so even though I started a context, I send the "not found" and FreeSWITCH will fallback to the static XML.
If someone knows how to do this more gracefully, I'd be appreciative.
(maybe in the dp_tranfer or dp_bridge, I can set GLOBAL $do++...)

## 6. there are various e.164 scripts, configured with the default array such as `array('local'=>9722, 'strip'=>1,'usa'=>1, 'israel'=>1, 'uk'=>1)`
`$p['local']` means it will prefix 9722 to 7 digit numbers (This should probably be pulled from the db)
`$p['strip']` means to strip +, 00, and 011 so that we have a uniform dialstring. 
I created a `$p['strip']=="normalize"` to change 00 and + to 011 if you'd prefer that. It *seems* unnecessary, but it's an option if you think otherwise.

usa/israel, uk is to convert "local" dialing from each region into the full string. e.g. make sure USA has a 1, israel e.g. 077-xxx-xxxx is 97277xxx-xxxx, and uk 0800xxx-xxxx? goes to 448... etc.
While these 3 countries DO coexist when dialed "locally", the countries you wish may not.
If you do write "local" dialplans for other countries, please share.




# SCHEMAS for MySQL
## a_lrn for lrn tracking:
```sql
CREATE TABLE IF NOT EXISTS `a_lrn` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `digits` bigint(11) unsigned NOT NULL,
  `new_npa` smallint(6) NOT NULL,
  `new_nxx` smallint(6) NOT NULL,
  `date_lrn` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
   PRIMARY KEY (`id`)
) ENGINE=MyISAM  DEFAULT CHARSET=latin1 AUTO_INCREMENT=0 ;
```
## a_did for incoming DID routing - use an extension number or prefix with sip:
```sql
CREATE TABLE IF NOT EXISTS `a_did` (
  `did_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `number` bigint(20) unsigned NOT NULL,
  `route` varchar(200) NOT NULL,
  PRIMARY KEY (`did_id`),
  UNIQUE KEY `number` (`number`)
) ENGINE=MyISAM  DEFAULT CHARSET=latin1 AUTO_INCREMENT=0 ;
```
## mod_lcr is mostly standard. Except:
1. I'm not querying most of the columns because I have no use for them.
2. I added tinyint for minimum and increment to "carriers", and moved the gateway there because I only have on for each.
So the table looks like:
```sql
CREATE TABLE IF NOT EXISTS `carriers` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `carrier_name` varchar(255) DEFAULT NULL,
  `enabled` tinyint(1) NOT NULL DEFAULT '1',
  `minimum` tinyint(3) unsigned NOT NULL,
  `increment` tinyint(3) unsigned NOT NULL,
  `prefix` varchar(100) NOT NULL,
  `suffix` varchar(100) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB  DEFAULT CHARSET=latin1 AUTO_INCREMENT=0 ;
```
