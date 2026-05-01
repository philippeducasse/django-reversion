  Bug 2 — Lines 225–237 in _save_revision: existence check still uses PK                                   
   
  When object_id_field is set, version.object_id holds a UUID (or whatever the custom field value is), not 
  the PK. But _save_revision does:
                                                                                                           
  model._base_manager.using(db).filter(pk__in=pks).values_list("pk", flat=True)

  It filters by pk__in=pks where pks are UUID values — so the filter silently returns nothing, and all     
  those versions get dropped before saving.
                                                                                                           
  This part needs to be made field-aware. One approach: for models using object_id_field, filter by that   
  field and return its values (not PKs), then compare against version.object_id as usual.
                                                                                                           
  Bug 3 — Line 199: get_for_object with ignore_duplicates                                                  
   
  previous_version = Version.objects.using(using).get_for_object(obj, model_db=model_db).first()           
                  
  get_for_object almost certainly filters by object_id=force_str(obj.pk) internally, which is wrong when   
  object_id_field overrides what object_id stores. You'd need to either fix get_for_object to accept an
  override, or do the lookup manually using content_type + object_id (which you've already computed at that
   point in the function).

  ---
  The immediate crash is bug 1 (obj[field]), bug 2 is a silent data loss (versions never saved), and bug 3
  only affects the ignore_duplicates path. Want me to fix all three? 