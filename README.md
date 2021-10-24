# fractional-indexing-utils
Common functionalities when using [fractional-indexing](https://github.com/rocicorp/fractional-indexing) npm package. Typescript supported.

Some weeks ago I had the necessity of allowing users to reorder their nested draggable lists, containing categories and items. I had basically the idea of creating the same as the fractional-indexing did, before I knew the already existence of it. As I am in a startup rush and time isn't my friend, I searched for dozens of pages at npm, in the hope that someone else had this idea before me and did implement it. In this case, someone had the idea and someone else implemented it.

While the fractional-indexing lib works, it lacks some built-in functionalities that I reused some times in my client and in my server, and certainly people that uses the fractional-indexing had to the something similar to what I've done.

For now, I will just drop its code here as I haven't befriended time yet. If you, fellow reader, found this somehow and want a npm package, open an issue so I feel more motivated to do it! I created this repo now because commonly I come to useful and reusable code but they never leave my programs. This is for now the middle ground between not publicizing at all and creating a npm package lol

It assumes your FI (fractional indexed) objects has the property 'sortId', containing the fractional-indexing generated key, a pattern that I came up with. It could be a dynamic key, but for now it suits me well.

When you want nested data, like categories and items, I suggest you to have them both as props in an object at the same level, and in the item, have the categoryId. This way you can easily change the item category and its order, without changing anything else. Smart! I had come up to this resolution because I am now using Firebase Firestore, and when both are in a single a doc, I can just deep update() the specific category/item with {['path.to.item']: 'notation'}.


```ts
import { generateKeyBetween } from 'fractional-indexing';


type obj = Record<string, unknown>;

type FiObjBase = {sortId: string};


/** Simple alphabetical ordering, how FI is designed to be sorted. */
function sortBySortId(a: FiObjBase, b: FiObjBase): number {
  if (a.sortId < b.sortId)
    return -1;
  if (a.sortId > b.sortId)
    return 1;
  return 0;
}


/** Returns a sorted array from a FI record. */
export function arrayFromFiRecord<T extends FiObjBase>(fiRecord: Record<string, T>): T[] {
  return Object.values(fiRecord).sort(sortBySortId);
}


export type ObjWithItsId<Obj extends obj> = Obj & {_id: string};


type NestedFiRecord<A extends obj, B extends obj, KeyChildren extends string> = ObjWithItsId<{
  [k in KeyChildren]: (ObjWithItsId<B>)[]
}> & A;


/** Constructs an array of arrays, from two FI objects, one that links to another one.
 * 
 * Both are FI-ordered. */
export function arrayFromNestedFiRecord<
  A extends FiObjBase,
  B extends FiObjBase & {[k in BToAProp]: string},
  BToAProp extends keyof B,
  KeyChildren extends string,
>(
  aFiRecord: Record<string, A>,
  bFiRecord: Record<string, B>,
  { bToAProp, keyChildren = '_children' as KeyChildren }: {
    /** What prop links 'b' to its parent 'a' */
    bToAProp: BToAProp;
    /** The prop in a that will contain b. */
    keyChildren?: KeyChildren;
  },
): NestedFiRecord<A, B, KeyChildren>[] {
  const record: Record<string, NestedFiRecord<A, B, KeyChildren>> = {};
  Object.entries(aFiRecord).forEach(([aId, aItem]) => {record[aId] = { ...aItem, _id: aId, [keyChildren]: [] } as NestedFiRecord<A, B, KeyChildren>;});
  Object.entries(bFiRecord).forEach(([bId, bItem]) => {
    const aItem = record[bItem[bToAProp]];
    if (aItem)
      aItem[keyChildren].push({ ...bItem, _id: bId });
  });
  return Object.values(record).map((aItem) => ({
    ...aItem,
    [keyChildren]: aItem[keyChildren].sort(sortBySortId),
  })).sort(sortBySortId);
}



export function getSortIdForIndex(fiArray: FiObjBase[], index: number, currentSortId?: string): string;
export function getSortIdForIndex(fiRecord: Record<string, FiObjBase>, index: number, currentSortId?: string): string;
/** From an array or a record of FI objects, it returns the sortId of the new item you want to add, at the index you want.
 * 
 * You may also "move" the item to a new index, by getting its new sortId. To do so,
 * enter the currentSortId of the item in question and its new index. */
export function getSortIdForIndex(data: (FiObjBase[] | Record<string, FiObjBase>), index: number, currentSortId?: string): string {
  if (typeof data === 'object')
    data = arrayFromFiRecord(data as Record<string, FiObjBase>);

  index = Math.min(index, data.length);
  index = Math.max(0, index);
  if (currentSortId)
    data = data.filter((i) => i.sortId !== currentSortId);
  // [0,1,2,3,4,5] - if I want to place between 1 and 2, the index would be 2. generateKeyBetween(a[1],a[2]), (index-1, index)
  // [0, 1] - if I want to move 0 after 1 (index 2), generateKeyBetween(a[1],null). We need to first remove the old position!
  return generateKeyBetween(data[index - 1]?.sortId ?? null, data[index]?.sortId ?? null);
}

```
