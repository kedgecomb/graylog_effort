-- Extractors are used to format or perform conversion on incoming messages
-- received by Graylog inputs.  The goal of extraction should be to extract 
-- and normalize enough data from the raw data contained in each message to 
-- enable querying the data, providing useable results.

-- Each input containing a different or new message format will require a new 
-- set of extractors, tailored to parse the data appropriately.

-- 