EXTENSION = pg_bao
MODULE_big = pg_bao
DATA = pg_bao--0.0.1.sql
OBJS = main.o 
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)

# Add PostgreSQL server include path
PG_CPPFLAGS += -I$(shell $(PG_CONFIG) --includedir-server)

# Add PostgreSQL library path
SHLIB_LINK = -L$(shell $(PG_CONFIG) --libdir)

#SHLIB_LINK = -lsqlite3
include $(PGXS)
