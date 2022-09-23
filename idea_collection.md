# Plain(er) SQL is better than JPA
* What are your reasons for using JPA?
    * The project already used it
    * It's a standard
* Preface
    * Totally possible, that many of my views are outdated due to my old
      knowledge of JPA. Let me know and I am more than happy to change my view.
* Why
    * Less hassle to setup (weak argument)
    * For microservices with just a few db entities it does not cost that much to
roll your own
    * No annotations anywhere
    * Matches good with DDD (IMHO).
    * Way less surprises aka Bad experiences (looking at outdated openJPA implementation)
    * Hibernate/JPA where meant for complex Object relations (find quote before
you include that)
        * https://www.javaperformancetuning.com/news/interview041.shtml
    * How often did you write your own SQL anyway? E.g. for a complex query?
    * Good luck doing JPA when you want to use coroutines
* What did we use
    * SpringJdbcTemplate
    * JdbcOperations from MicronautData 
* How?
    * Load all data you are going to need at once in the beginning
    * Do all changes to business code (collect those)
    * Store all changed objects to db at the end
    * UPSERTs are your friend
    * No more db access right in the middle of your business logic
    * You can do that with JPA, too. But at what cost?
* UPSERTs!!!!
    * private const val UPSERT_STATEMENT =
    """
INSERT INTO vehicle_authorization (fin, authorization_owner_id, authorization_owner_type, vehicle_owner, authorization_id, authorization_type, created_at, last_updated_at, locking_version)
VALUES (?, ?, ?, ?, ?, ?, ?, ?, 1)
ON CONFLICT (fin, authorization_owner_id, authorization_owner_type) DO UPDATE
  SET authorization_id = excluded.authorization_id,
      authorization_type = excluded.authorization_type,
      vehicle_owner = excluded.vehicle_owner,
      last_updated_at = excluded.last_updated_at,
      locking_version = vehicle_authorization.locking_version + 1
WHERE vehicle_authorization.locking_version = ?
    """
@Transactional
    override fun store(authorization: Authorization) {
        jdbcOperations.insert(
            UPSERT_STATEMENT,
            {
                setAuthorizationParameter(authorization)
            }
        ) {
            if (it == 0) {
                throw OptimisticLockingException(
                    "Optimistic Locking Failure when updating Authorization: " +
                        authorization.fin + " " + authorization.owner.id + " " +
                        mapAuthorizationOwnerType(authorization.owner.type)
                )
            }
        }
    }
    * vs load from the database
    * Datadog screenshot
* Downsides
    * If you have a complex Object relation it is a pain in the ass (Think of
      ARS)
    * 
* What didn't we implement?
    * Pre-Generating Sequence values
* What I would love to use?
    * jooq
